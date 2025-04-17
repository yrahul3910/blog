+++
date = '2025-04-16T20:23:29-04:00'
draft = false
title = 'ZQA Devlog 01'
+++

# Introduction

This is the first in a series of posts logging the development of `zqa`, a RAG tool I am working on to work with Zotero. The eventual goal is that you will be able to, from your terminal (as an avid terminal user myself), ask questions and have LLMs respond with citations from your Zotero library. I also want this tool to help you find things: so maybe you remember certain details about a paper but don't quite know what it was about or its title--a problem that has become all too real with my 1,300-item library. Why is it called `zqa`? Because I'm not creative, and "Zotero Q & A" was the first thing that popped into my head. Also, the letters in `zqa` happen to be very close, so it's easy enough to type.

I will be writing it in Rust, for a few reasons. First, it will avoid me having to Rewrite It In Rust later. Jokes aside, the benefits are twofold: I get to learn Rust by writing a real project in it, and second, because Rust has fewer high-level libraries that do everything for you, I will be forced to learn the details of how things work, which I think is very important for devs. Initially, I did start writing it in Python, but I grew increasingly frustrated with the lack of type enforcement by default and how I was not forced to think about low-level ideas like memory management.

I will be showing plenty of code, but it is not the full code. You can see that on [the GitHub repo](https://github.com/yrahul3910/zotero-rag/).

# Getting the library

Zotero stores information about the library in a SQLite file. We can read this, but we first need to find it.

```rs
fn get_lib_path() -> Option<PathBuf> {
    match env::consts::OS {
        "linux" | "macos" | "windows" => {
            UserDirs::new().map(|user_dirs| PathBuf::from(user_dirs.home_dir()).join("Zotero"))
        }
        _ => None,
    }
}
```

Straightforward enough. Inside the directory returned by this function is a `zotero.sqlite` that we need to read. Now, we need to parse the info we need out of this table. After some spelunking through the table schemas, I decided I want titles, abstracts, locations of PDF attachments, and notes. A clever SQL `CASE WHEN` clause allows us to do this in one query:

```rs
/// Parses the Zotero library. If successful, returns a list of metadata for each item.
pub fn parse_library() -> Result<Vec<ZoteroItemMetadata>, LibraryParsingError> {
    if let Some(path) = get_lib_path() {
        let conn = Connection::open(path.join("zotero.sqlite"))?;

        // The SQL query essentially combines the titles and abstracts for each paper into a
        // single row. This is done using GROUP BY on the keys, and MAX to get the field itself.
        let mut stmt = conn.prepare(
            "SELECT items.key AS libraryKey,
                MAX(CASE WHEN fieldsCombined.fieldName = 'title' THEN itemDataValues.value END) AS title,
                MAX(CASE WHEN fieldsCombined.fieldName = 'abstract' THEN itemDataValues.value END) AS abstract,
                itemNotes.note AS notes,
                itemAttachments.path AS filePath
            FROM items
            INNER JOIN itemData ON items.itemID = itemData.itemID
            INNER JOIN fieldsCombined ON itemData.fieldID = fieldsCombined.fieldID
            INNER JOIN itemDataValues ON itemData.valueID = itemDataValues.valueID
            INNER JOIN itemAttachments ON items.itemID = itemAttachments.itemID
            LEFT JOIN itemNotes ON items.itemID = itemNotes.itemID
            WHERE fieldsCombined.fieldName IN ('title', 'abstract')
            GROUP BY items.key;
        ")?;

        let item_iter: Vec<ZoteroItemMetadata> = stmt
            .query_map([], |row| {
                let res_path: String = row.get(4)?;
                let split_idx = res_path.find(':').unwrap_or(0);
                let filename = res_path.split_at(split_idx).1;
                let lib_key: String = row.get(0)?;

                Ok(ZoteroItemMetadata {
                    library_key: lib_key.clone(),
                    title: row.get(1)?,
                    paper_abstract: row.get(2).unwrap_or_default(),
                    notes: row.get(3)?,
                    file_path: path.join("storage").join(lib_key).join(filename),
                })
            })?
            .filter_map(|x| x.ok())
            .collect();

        Ok(item_iter)
    } else {
        Err(LibraryParsingError::LibNotFoundError)
    }
}
```

Good, we now have a way to get metadata from the library, including, most importantly for RAG, the locations to the PDF files. We would now like to proceed by storing embeddings of these in a vector database, but there's a more fundamental issue: how do you parse a PDF file? In Rust, libraries for PDFs don't automagically do this for you--and that is a *strength*, in my opinion. We get to learn how PDFs work!

# PDF Parsing

We're at the most exciting part! First, let's understand how the PDF format works. A PDF file has its content as a sequence of bytes, and it has embedded information about things like fonts. As we will see, fonts in PDFs are triply-nested dictionaries--but that's for a later time. First, let's read the file itself.

```rs
pub fn extract_text(file_path: &str) -> Result<String, Box<dyn Error>> {
    let doc = Document::load(file_path)?;
    let mut parser = PdfParser::with_default_config();

    let content = doc
        .page_iter()
        .map(|page_id| {
            parser
                .parse_content(&doc, page_id)
                .unwrap_or("".to_string())
        })
        .collect::<Vec<_>>()
        .join("");

    Ok(content)
}
```

We are using the `lopdf` crate to help here. This crate gives us a more structured view into the PDF. It gives us a way to access the content as a byte stream as well as reading embedded metadata such as fonts through a function. We now need to write the `PdfParser` struct and implement its `parse_content` function.

First, let's understand how PDFs even work. The content of a PDF is a sequence of bytes containing what are essentially commands to the renderer. You can think of these commands as telling the renderer what to do. In general, at least for simple PDFs that don't [implement Doom](https://github.com/ading2210/doompdf), a renderer should be able to use a single pass to display the contents. The set of commands and their argument structure is nicely laid out in a [230 page document by Adobe](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/pdfreference1.0.pdf). We will not be reading that, obviously. Instead, we will take inspiration from [this blog post](https://adventures.michaelfbryan.com/posts/parsing-pdfs-in-rust/), which gives us the basics. I highly recommend this.

It turns out academic PDFs look mostly like this. We will also use the fact that at least for CS, most academics use LaTeX to typeset their documents. Which means that we can create simple PDF files and look at the resulting content bytes. Upon doing this, we notice that content is put inside `TJ` commands, which basically contain arrays of content instead of individual pieces of content like `Tj` does. We now know the first step:

```rs
let content = doc
    .get_page_content(page_id)
    .map_err(|_| PdfError::ContentError)?;
let content = String::from_utf8_lossy(&content).to_string();
let mut rem_content = content;
let mut parsed = String::new();

loop {
    if !rem_content.contains("TJ") {
        break;
    }
    // Parse
}
```

Inside the `TJ` command, we have words or subwords in parentheses, separated by numbers which tell the renderer how much to move the cursor (if you're old enough to remember [Logo](https://en.wikipedia.org/wiki/Logo_(programming_language)), similar idea). Let's parse these.

```rs
/* Here's our strategy. We'll look for pairs of (), consuming words inside.
 * Then, we'll consume an integer. If that integer is less than SAME_WORD_THRESHOLD, the next
 * chunk will be appended to the current word. Otherwise, we add a space. */
while cur_content.contains('(') {
    let idx1 = cur_content.find('(').ok_or(PdfError::ContentError)?;
    let idx2 = cur_content.find(')').ok_or(PdfError::ContentError)?;

    if idx1 >= idx2 {
        break;
    }

    if !cur_content[idx2..].contains('(') {
        parsed += " ";
        break;
    }

    let idx3 = cur_content[idx2..].find('(').unwrap() + idx2;
    let spacing = cur_content[idx2 + 1..idx3].parse::<i32>().unwrap().abs();

    if !(0..=self.config.same_word_threshold).contains(&spacing) {
        parsed += " ";
    }

    cur_content = &cur_content[idx2 + 1..];
}

rem_content = rem_content[end_idx + 2..].to_string();
}
```

And with that, we have our basic parser! It has several issues, so let's go over them one at a time. The most obvious one is that sometimes, LaTeX uses unnormalized Unicode to represent code points. There's also the issue of ligatures that some fonts support. We need to take care of these.

```rs
/// A lazy-loaded hashmap of octal character replacements post-parsing.
/// Some of these come across because of ligature support in fonts. This
/// is not exhaustive, however.
static OCTAL_REPLACEMENTS: Lazy<HashMap<&str, &str>> = Lazy::new(|| {
    let mut m = HashMap::new();
    m.insert("\\050", "(");
    m.insert("\\051", ")");
    m.insert("\\002", "fi");
    m.insert("\\017", "*");
    m.insert("\\227", "--");
    m.insert("\\247", "Section ");
    m.insert("\\223", "\"");
    m.insert("\\224", "\"");
    m.insert("\\000", "-");

    m
});

// Parse the weird octal representations
for (from, to) in OCTAL_REPLACEMENTS.iter() {
    parsed = parsed.replace(from, to);
}
```

# Fin

There are a few more issues, but we will get to those in a future post. This is it for the first devlog. I will write more posts detailing how different components work as I write them.

---
author: Taylor Deckard
title: "Creating a Vim Plugin to Edit Parquet Files"
date: 2025-05-04
description: >- 
tags:
    - programming
    - rust
    - vim
    - spark
    - iceberg
---

For the past several years, my work has focused on data engineering and developing tools that simplify data processing. One of the most common file formats I work with is Parquet, a columnar storage file format that is optimized for use with big data processing frameworks like Apache Spark and Apache IcebergFor the past several years, my work has focused on data engineering and developing tools that simplify data processing..

In my most recent project, I have worked on building end-to-end testing capabilities for data pipelines. To do this, input data is generated and stored in Parquet files within the GitHub repository. After the data has been sent through a pipeline, another "expected output" Parquet file is compared against the actual output.

For anyone unfamiliar, Parquet files are binary (not human-readable,) which makes it difficult to inspect the contents of these files or to make quick edits. While there are tools available to convert Parquet files to CSV or JSON for easier inspection, I wanted to create a Vim plugin that would allow me to view and edit Parquet files directly within Vim.

## The Idea

With Vimscript, it is possible to run code during certain events. For example, when a file is opened, you can run a function to check the file type and load the appropriate syntax highlighting. Similarly, there is another hook that allows you to run code when a file is written.

So, using these hooks I can convert a parquet file to some easier to edit format (I chose [JSONL](https://jsonlines.org/)) when the file is opened, and then convert it back to Parquet when the file is saved.

## Starting with Vimscript

The Vimscript for this plugin is relatively simple.

### Viewing Parquet Files

Here is the part that handles viewing the contents of a Parquet file:

```vim
function! ParquetView(filepath)
  let tmpfile = tempname() . ".vimparq.jsonl"
  let cmd = "vimparq view " . shellescape(a:filepath) . " > " . shellescape(tmpfile)
  call system(cmd)
  silent! execute 'edit ' . tmpfile
  setlocal filetype=json
  setlocal nowrap
  setlocal syntax=off
  let b:parquet_original = a:filepath
  let b:parquet_tmpfile = tmpfile
endfunction
```

When the `ParquetView` function is invoked with a file path (`filepath`), it generates a unique temporary filename with a `.vimparq.jsonl` extension. It then constructs and runs a shell command using an external tool (`vimparq view`, more on this in a bit) to convert the given Parquet file to JSONL format, redirecting the output to the temporary file. After the command completes, the function opens the resulting JSONL file in the current Vim buffer, sets the filetype to JSON, disables line wrapping, and disables syntax highlighting for better performance or cleaner viewing. Finally, it stores the original Parquet file path and the temporary JSONL path in buffer-local variables (`b:parquet_original` and `b:parquet_tmpfile`) for later use, such as saving changes back to the Parquet format.

The hook that calls this function is `BufReadPost`. This is a Vim event that is triggered after a buffer has been read. The following code registers the `ParquetView` function to this event:

```vim
autocmd BufReadPost *.parquet call ParquetView(expand("%:p"))
```

This line sets up an autocommand that listens for the `BufReadPost` event specifically for files with the `.parquet` extension. When such a file is opened, it calls the `ParquetView` function, passing the full path of the file being opened (`expand("%:p")`) as an argument.

One more thing to note: the temporary file has a `.vimparq.jsonl` extension for good reason. This is important because it allows us to differentiate between the temporary JSONL file and any other JSONL files that might be opened in Vim.

### Editing Parquet Files

Now for the saving part. The `BufWriteCmd` event is triggered just before a buffer is written to a file. This is where we will convert the JSONL file back to Parquet format:

```vim
function! ParquetSave(filepath)
  if exists("b:parquet_tmpfile") && exists("b:parquet_original")
    echom "🛠 ParquetSave triggered"
    " Save buffer contents to the temp file
    write!
    let cmd = "vimparq edit " . shellescape(b:parquet_original) . " " . shellescape(b:parquet_tmpfile)
    let output = system(cmd)
    echom "Updated Parquet: " . b:parquet_original
    " Optionally clean up
    call delete(b:parquet_tmpfile)
    " Reload original .parquet if needed
  else
    echom "No edit buffer associated!"
  endif
endfunction
```

This function checks if the buffer-local variables `b:parquet_tmpfile` and `b:parquet_original` exist. If they do, it writes the current buffer contents to the temporary JSONL file, constructs a command to convert the JSONL back to Parquet format using the external tool (`vimparq edit`), and executes that command. After the conversion, it cleans up by deleting the temporary JSONL file. If the buffer-local variables do not exist, it outputs a message indicating that no associated temporary edit buffer was found.

Again, this function is registered to the appropriate event:

```vim
autocmd BufWriteCmd *.vimparq.jsonl call ParquetSave(expand("%:p"))
```

### Creating Parquets from Scratch

So far, we've only converted existing Parquet files to JSONL and back. But what if you want to create a new Parquet file from scratch? This is also possible with one more function:

```vim
function! ParquetCreate(filepath)
  " Called when a new *.parquet file is opened
  let tmpfile = tempname() . ".vimparq.jsonl"
  silent! execute 'edit ' . tmpfile
  setlocal filetype=json
  setlocal nowrap
  setlocal syntax=off
  let b:parquet_original = a:filepath
  let b:parquet_tmpfile = tmpfile
endfunction
```

This function is similar to the `ParquetView` function, but it does not run the external command. Instead, it creates a new temporary JSONL file and opens it in Vim for editing. The user can then write JSONL data directly into this buffer. When they save the buffer, the `ParquetSave` function will be called to convert the JSONL data into a Parquet file.

### What is vimparq?

Aside from being the name of the repository, "vimparq" is also the name I have given to the command-line tool that handles the conversion between Parquet and JSONL formats. The Vim plugin itself is just a wrapper around this tool, which is written in Rust.

## Writing the Parquet Converter CLI

I wrote a very simple Rust program with [Clap](https://docs.rs/clap/latest/clap/) and [arrow](https://arrow.apache.org/rust/arrow/index.html). Clap is a framework for command-line argument parsing, and Arrow is a Rust implementation of the Apache Arrow columnar data format, which is the underlying format for Parquet files.

Here is the code for the `view` command:

```rust
use std::fs::File;
use std::path::PathBuf;
use arrow::record_batch::RecordBatch;
use arrow::json::LineDelimitedWriter;
use parquet::arrow::arrow_reader::ParquetRecordBatchReaderBuilder;

pub fn view_parquet(path: &PathBuf) -> Result<(), Box<dyn std::error::Error>> {
    let file = File::open(path)?;
    let builder = ParquetRecordBatchReaderBuilder::try_new(file)?;
    let reader = builder.build()?;
    let batches: Vec<RecordBatch> = reader.collect::<std::result::Result<_, _>>()?;

    write_json_lines(&batches)
}

fn write_json_lines(batches: &[RecordBatch]) -> Result<(), Box<dyn std::error::Error>> {
    // Lock stdout (or use any Write impl—File, Vec<u8>, etc.)
    let stdout = std::io::stdout();
    let mut handle = stdout.lock();

    // Create a JSON-Lines writer over that handle
    let mut writer = LineDelimitedWriter::new(&mut handle);

    // Write each RecordBatch
    // The write_batches method takes &[&RecordBatch]
    let batch_refs: Vec<&RecordBatch> = batches.iter().collect();
    writer.write_batches(&batch_refs)?;
    
    // Finish to flush any buffered output
    writer.finish()?;

    Ok(())
}
```

Given a Parquet file path, this function opens the file, reads it into a vector of `RecordBatch` objects, and then writes each batch in JSONL format using the `LineDelimitedWriter`. The `write_json_lines` function handles the actual writing of the JSONL data to standard output.

The Vim plugin will load this output into a buffer for viewing and editing. 

The `view` command is just one of the commands in the `vimparq` tool. There is also a `edit` command that takes a JSONL file and writes it back to Parquet format. Arguments supplied to this command are the path to the JSONL file and the path to the output Parquet file. 

```rust
use std::fs::File;
use std::io::{BufReader, Seek, SeekFrom};
use std::path::PathBuf;
use std::error::Error;
use std::sync::Arc;

use arrow::json::reader::{ReaderBuilder, infer_json_schema_from_seekable};
use parquet::arrow::ArrowWriter;

pub fn edit_parquet(path: &PathBuf, json_path: &PathBuf) -> Result<(), Box<dyn Error>> {

    // Open JSONL file
    let json_file = File::open(json_path)?;
    let mut buf_reader = BufReader::new(json_file);
    buf_reader.seek(SeekFrom::Start(0))?;

    // Infer schema
    let (schema, _) = infer_json_schema_from_seekable(&mut buf_reader, Some(1))?;
    let schema = Arc::new(schema);

    // Read JSON using the schema
    let builder = ReaderBuilder::new(schema.clone());
    let mut json_reader = builder.build(buf_reader)?;

    // Overwrite Parquet file
    let out_file = File::create(path)?;
    println!("Writing to: {:?}", out_file);
    let mut writer = ArrowWriter::try_new(out_file, schema, None)?;

    while let Some(batch_result) = json_reader.next() {
        let batch = batch_result?;
        writer.write(&batch)?;
    }

    writer.close()?;
    Ok(())
}
```

This function opens the JSONL file, infers its schema, and then writes it to the specified Parquet file. The `ArrowWriter` is used to handle the writing of the data in Parquet format.

## Wrapping Up

This Vim plugin allows you to view and edit Parquet files directly within Vim, making it easier to inspect and modify data without needing to convert it to a different format first. The `vimparq` command-line tool handles the conversion between Parquet and JSONL formats, while the Vim plugin provides the editing experience. You can find all of the code for this project as well as installation instructions [on GitHub](https://github.com/taylordeckard/vimparq).

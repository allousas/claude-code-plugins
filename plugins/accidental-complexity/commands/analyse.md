---
allowed-tools: *
argument-hint: [files to analyse]
description: Analyse accidental complexity
model: sonnet
---

{{#if $1 is empty}}
**Stop immediately.** 
Inform the user with the following message: 
"No files provided for analysis. Please provide a list of files or a pattern to select files for analysis.
Keep in mind that analysing the entire codebase at once could be too expensive 💸, please provide a list of files or a pattern (up to 50 max)."

{{else}}
First, run file-selector subagent to generate a detailed queue of files to analyse.

Second, if the resulting queue file contains more than 50 files, inform the user with the following message and ask for if it is ok to proceed:
"⚠️ Too many files to analyse (X files). This could be too expensive 💸💸💸 and take a long time ⏱️⏱️⏱️. Do you want to proceed with analysing all files? (yes/no)"

Then, if user wants to proceed, for each entry in the resulting queue file, run sequentially, file by file, run just the first 10 files:
    - Run single-file-accidental-complexity-analyzer subagent with the file
    - Mark file as 'analysed' in the queue file when subagent finishes
    - Don't create a new queue file, just update the existing one by changing the status of each file to "analysed" once processed
    - Don't append the findings to any file in this command, the single-file-accidental-complexity-analyzer subagent is responsible for writing findings to `accidental-complexity-findings.jsonl`
    - Don't reprocess any files marked as "analysed" in the queue file

Finally, after processing all the files (all queue entries marked as "analysed"), run report-generator subagent to generate the final report.
{{/if}}
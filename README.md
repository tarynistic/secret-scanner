# secret-scanner
Collaborative project for a software engineering project. A command-line tool will be developed to detect any API keys, tokens, or credentials in code files to combat security issues. Any output with hardcoded keys or other confidential information is consequently printed as a hashed out string with asterisks for further information security.


#note:
  any edits of file names will need to be updated in scripts that reference those files.

# Steps to Run
On macOS, `python3 -m secret_scanner` runs the project. The target rust file should exist just outside the secret_scanner directory as long as they're existing within the same folder 
### *Example*
`"Secret Scanner"/secret_scanner` and `"Secret Scanner"/vulnerable_sample.rs`.
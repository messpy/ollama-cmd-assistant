# olm — One-Look Model  
A lightweight **LLM-powered CLI analyzer** for commands, files, and environments.

---

## Overview

`olm` is a minimal, safe, and fast **LLM assistant for the terminal**.  
It explains shell commands, analyzes files, inspects the current environment,  
and suggests next steps — all without executing anything dangerous.

`olm` behaves like `git`:  
**simple command, automatic mode detection, powerful output.**

---

## Features

### 1. Automatic Mode Detection

| Input Type        | Behavior                                      |
|-------------------|------------------------------------------------|
| *(no arguments)*  | Analyze current environment (“now mode”)       |
| File path         | Analyze file structure & purpose               |
| Other text        | Treat as shell command and explain it          |

---



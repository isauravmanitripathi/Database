# Narration Output Structure

The `narration.json` file contains a sequence of steps used to generate an automated coding tutorial or narration. Each object in the array represents a specific action, such as creating a file/folder or highlighting a block of code with an accompanying explanation.

## Structure

The file is a JSON Array of objects. There are three main types of actions:

### 1. Create Folder
Represents the creation of a directory.
```json
{
  "create-folder": "path/to/folder"
}
```

### 2. Create File
Represents the creation of a file.
```json
{
  "create-file": "path/to/file.ext"
}
```

### 3. Narrate Code
Opens a file, scrolls to a specific range, highlights the code, and provides a narration script.

| Field | Description |
| :--- | :--- |
| `open-file` | Relative path to the file being explained. |
| `range` | Object defining `start` and `end` positions (line and character) for highlighting. |
| `code` | The actual code snippet being discussed. |
| `narration` | The text explanation to be read aloud or displayed. |

## Example Entry

Here is a sample entry from `narration.json`:

```json
{
  "open-file": "dev/gen_synthetic_data.py",
  "range": {
    "start": {
      "line": 9,
      "character": 0
    },
    "end": {
      "line": 9,
      "character": 13
    }
  },
  "code": "load_dotenv()",
  "narration": "This line loads environment variables from a .env file into the current process environment, which is a standard pattern for managing configuration and sensitive credentials outside of source code. In the context of this file, which generates synthetic conversational data for training and evaluation, this initialization step ensures that any API keys or configuration values the script needs are available at runtime."
}
```

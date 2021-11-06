# Cisco CLI Tips and Tricks

## Key Combinations

### Recalling Commands

|                |                                         |
| -------------- | --------------------------------------- |
| Ctrl+P or UP   | Moves backwards through command history |
| Ctrl+N or DOWN | Moves forward through command history   |

### Moving Cursor

| Key Combination | Result                                   |
| --------------- | ---------------------------------------- |
| Ctrl+B or LEFT  | Moves 1 character to the left (Backward) |
| Ctrl+F or RIGHT | Moves 1 character to the right (Forward) |
| ESC,B           | Moves 1 word to the left (Backward)      |
| ESC,F           | Moves 1 word to the right (Forward)      |
| Ctrl+A          | Moves to the beginning of the line       |
| Ctrl+E          | Moves to the end of the line             |

### Deleting Entries

| Key Combination  | Result                                                                    |
| ---------------- | ------------------------------------------------------------------------- |
| DEL or BACKSPACE | Deletes one character to the left of the cursor                           |
| Ctrl+D           | Deletes the character at the cursor                                       |
| Ctrl+K           | Deletes all the characters from the cursor to the end of the command line |
| Ctrl+U or Ctrl+X | Deletes all the characters from the beginning of the line to the cursor   |
| Ctrl+W           | Deletes the word to the left of the cursor                                |
| ESC,D            | Deletes from the cursor to the end of the word                            |

### Recalling Deleted Entries

| Key Combination | Result                                                                        |
| --------------- | ----------------------------------------------------------------------------- |
| Ctrl+Y          | Recalls the most recent characters deleted with Ctrl+X, Ctrl+K or Ctrl+U      |
| ESC,Y           | After recalling with Ctrl+Y, browse through the history of deleted characters |

### Other

|                  | Capitalizes the letter at the cursor                                                                                          |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Ctrl+L or Ctrl+R | Transposes the character to the left of the cursor with the character to the right                                            |
| Ctrl+T           | Transposes the character to the left of the cursor with the character to the right                                            |
| ESC,C            | Capitalizes the letter at the cursor                                                                                          |
| ESC,L            | Changes the word at the cursor to lowercase                                                                                   |
| ESC,U            | Changes the word at the cursor to uppercase                                                                                   |
| Ctrl+V or ESC,Q  | Make the system accept the following key as a command, not asa an editing command – Useful for inserting “?” into the command |

## Limiting Output

When running a show command you can limit the output by only showing the lines that begin with, include or do not include a certain Regular Expression:

```
R# show COMMAND | {being|include|exclude} REGEX
```

When the output is long enough to generate a **-More-** line, you can filter the output using :

```
-More-
/REGEX
! Goes to the first match of Regular Expression
-REGEX
! Displays lines that do not contain the Regular Expression
+REGEX
! Displays lines that contain the Regular Expression
```

### Regular Expressions

Regular expressions define patterns of characters that are used to match lines in a text entry. Usually a character in the REGEX matches itself in a string, but there are some special characters or groups of characters that don’t match themselves:

| Character | Meaning                                                                                                                  |
| --------- | ------------------------------------------------------------------------------------------------------------------------ |
| .         | Matches a single character, including space                                                                              |
| \_        | Matches: , { } ( \_ space beginning-of-string end-of-string                                                              |
| \\        | Escape character used to match a special character                                                                       |
| Groups    |                                                                                                                          |
| \[LIST]   | Matches any character in the LIST. The LIST can be specified as an unordered group of characters and ranges, like a-dA-D |
| \[^LIST]  | Matches any character that is not part of the LIST.                                                                      |

A pattern can be a character or a group of characters between parenthesis (). Between parenthesis, a | acts as a logical OR. Patterns can be multiplied if followed by one of the Multiplier characters. Also, you can recall a pattern used once, if you use a \ followed by the number which indicates which pattern it is.

| (PATTERN)            | Matches the PATTERN                                                  |
| -------------------- | -------------------------------------------------------------------- |
| (PATTERN1\|PATTERN2) | Matches PATTERN1 or PATTERN2                                         |
| Multipliers          |                                                                      |
| \*                   | Matches 0 or more sequences of the pattern                           |
| +                    | Matches 1 or more sequences of the pattern                           |
| ?                    | Matches 0 or 1 sequences of the pattern                              |
| ^                    | Matches the beginning of string                                      |
| $                    | Matches end of the string                                            |
| \n                   | Recalls the n-th pattern and uses it again in the regular expression |

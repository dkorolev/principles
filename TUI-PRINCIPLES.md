# Terminal board principles

> A living document. `ENG-PRINCIPLES.md` §2 and §16 state how a command talks to a person and to a program. This file is the screen a person walks. The house maxim — **every operation is on the board** — is enforced here. Each principle is an imperative. Product features appear only as examples. Section references here (§N) name this file; references into the sibling are spelled `ENG-PRINCIPLES §N`.

---

## 1. The maxim

- **Every operation a person can run is on the board.** They reach it with the arrow keys and Enter. They do not have to already know the spelling. A value the service already has is a list of names. When that list is a command of the account, the board fetches it first, shows a spinner, and then offers the names. A program still runs one command and exits (`ENG-PRINCIPLES` §16); the board is the person's way through the same catalog.

- **Up and down move. Right or Enter opens. Left or Esc steps back.** A service opens onto the accounts this person may use. An account opens onto its commands. Search lists matching names first and matching help under that, and a name match is not repeated.

- **The board is not a shell and not a prompt.** It does not ask a question and wait. It does not read a script. Each line it runs is still one command, checked the same way as if the person had typed it.

## 2. A known value is a list of names

- **If the service already has the value, the person selects it.** A key, a person, an account, a lease, a level (`read|write`): the field lists those names. Up and down move through the list. Typing narrows it. Enter uses the highlighted name. They should not have to type a value the service is holding.

- **Select by the name, submit what the command needs.** A key is shown as the person's name. The command receives the key. A passkey is shown as the name the person gave it. The command receives its id. The row and the value may differ; the row is what a person can recognize.

- **What they type is sent when it matches no name.** A custom value is allowed. The command then says what was wrong. The list does not trap them into a value that exists.

- **Typing starts a search. It does not extend a name they already selected.** Once the box holds a highlighted name, the next character narrows the list from that character. Backspace edits the text and narrows again.

- **An open value stays a field.** A message, a new secret, a key being added for the first time: there is no list, because the service does not have it yet. When a list does exist beside that field, it is the values already stored, and a typed value that matches none of them is still sent.

- **A field that an account can list is loaded before it is offered.** On a command of an account (`slack work post`), a field whose name is the list (`channels` fills `CHANNEL`) runs that account's list command first. The board shows a spinner while the list is fetched, then offers the names. The same list is kept for the rest of the session. Each row of the list document has `name`, and that string is the choice. An empty or failed list leaves the field as one the person can type. The fetch is the list command, checked like any other command.

## 3. Do not draw an empty negative

- **An empty list of what the person cannot do is not shown.** If every command can run, there is no "cannot run" heading, column, or zero. A heading for nothing reads as a problem. The same rule covers any other empty negative: no "0 blocked", no "nothing denied", no disabled section that explains an absence.

- **When something cannot run, say so, with the reason, and keep it.** A blocked command stays visible, with a plain reason (`needs write`, `not in your grant`). It does not vanish, and it does not appear when the list is empty.

## 4. SSH-powered board

- **`ssh HOST` from a terminal, with no command, is the board.** The connection stays open. The person walks the catalog and runs commands. A named command (`ssh HOST slack`) still runs once and closes. A program, or a connection without a terminal, is not given the board: it gets help, or the one command it named.

- **One signature, one session.** The board does not open a shell and it does not hand the line to a shell. There are no expansions, no substitutions, and no comments (`ENG-PRINCIPLES` §16). `#general` is a channel name.

- **Each command the board runs is still a command.** Grants, dry-run, and confirmation are checked when it runs, not when the room was opened. The board is a way to build the command. It is not a way around the check.

- **The server paints frames. It does not put its own terminal into raw mode.** Keystrokes, resizes, and close come from the client's terminal. The board restores that screen when the person leaves. A resize redraws the same place.

- **Output stays relative.** A command the board shows is `slack work whoami`, never the `ssh` line that reached the host (`ENG-PRINCIPLES` §16). The person already has the connection.

- **The host offers the values it holds.** The board itself does not know what a key is. The host, which has the people, the keys, the accounts, and the leases, attaches those lists to the fields. A board opened against a catalog that still says `ACCOUNT` asks for the account, and lists the real accounts when the host knows them.

## 5. One calm screen

- **The cursor sits in the field being typed.** A password, a secret, or a token shows asterisks. The value that runs is what was typed.

- **Color is the product's existing palette.** Do not invent a new success color for the board. A mark can show which row is selected. The word of the command stays readable.

- **A long command stays one line.** The quoted line under the form is what Enter will run, so a paste into a shell keeps each argument. A value wider than the window is cut with an ellipsis; it is not wrapped onto the next row.

- **Nothing flashes.** A key rewrites the rows that changed. Opening a room does not redraw the whole catalog from scratch once the catalog is loaded. A command that is running says so before its answer replaces the list.

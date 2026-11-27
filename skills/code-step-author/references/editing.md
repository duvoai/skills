# Edit a code step's program

You change a code step's program yourself: read its files, write the new
content, and save it to a draft revision. This works the same for a
single-file program (one file, `main.py`) and a multi-file one. Do not send a
code step to `generateRevision`.

The tools: `listAgentRevisions`, `getRevision`, `listCodeStepFiles`,
`getCodeStepFile`, `uploadCodeStepFile` and `updateRevision`, with
`createRevision` when the step has no draft. Most Duvo tools load on demand; find
these by name with tool search when they are not loaded yet.

## 1. Agree on the change

A request for advice is not a request to save. Before you write, say in one or
two sentences what the change will do and which files it touches, and get the
user's yes the way your host confirms a revision save. In Ask Duvo that is the
question card the chat's editing rules describe; a host with no such rule
takes a clear request ("make it skip invoices under 100 EUR, and save it") as
the yes. Ask first when the change is ambiguous.

## 2. Find the revision to edit

Save only to a draft revision. A save to the live revision changes what runs on
the next Case. `require_draft: true` on every save (section 5) makes sure it
cannot.

On a team with automation revisions, an edit lands in an Automation draft
revision. When the user has one open or names one, pass its id as
`selected_automation_revision_id` on every `updateRevision` and
`createRevision`, retries included; without it the edit goes to the
Automation's open draft: the active revision's draft, or, on an Automation
that has never been made active, its existing draft. Do not invent the id.

1. **Find the draft through the live revision.** `listAgentRevisions` for the
   step with `status: "live"`, then `getRevision` on it. Do not build your
   change yet: the draft can hold files and changes the live revision does
   not. Send `updateRevision` to the live revision with its `config` exactly as
   you read it, `expected_config_hash` set to its `config_hash`, and
   `require_draft: true`. This save changes no content. What it does tells
   you which revision to edit:
   - **It saves.** The team has automation revisions. The server never saves
     to the live revision: it sent the save to this step's draft in the
     selected Automation revision, or in the active revision's draft when none
     is selected, and forked that draft from the live revision when the step
     had none there. The revision id the save returns is the one to edit.
   - **409 `revision_config_changed`.** The step's draft already holds other
     changes. Nothing was saved. The refusal's `build_id` is the revision to
     edit.
   - **409 `revision_not_draft`.** The team saves to the step's own draft.
     Nothing was saved. Go on with 2.
   - **Any other refusal:** handle it as section 5 says.
2. **Without automation revisions, or with no live revision** (a step that
   has never been made live, such as one in an Automation that has never been
   made active, or one added in an Automation draft): `listAgentRevisions` for
   the step, and take the draft (`status: "draft"`). With several drafts, take
   the one the user named or has open; otherwise the most recent one. If the step has neither a
   live revision nor a draft, tell the user and stop. With no draft but a live
   revision, create one from the live revision with `createRevision`, passing `source_build_id` (the
   live revision's id) and the config you read from it with `getRevision`.
   On a team with automation revisions, when the step has no live revision,
   the draft you took may be left over from a discarded Automation draft. Any
   save to it adds the step, with its old program, to an Automation draft.
   That is the selected one (when the selected revision is the active one,
   its draft, which the save creates if there is none); with none selected,
   it is the Automation's open draft: the active revision's draft, which the
   save creates if there is none, or, on an Automation that has never been
   made active, its existing draft. When the save would create the active
   revision's draft but the Automation already has an open draft based on an
   older revision, it is refused with a conflict and writes nothing: tell
   the user to update or discard that draft in Automations, then edit again.
   No read tells you which steps an Automation draft holds, so unless
   the user is editing the step in that Automation draft right now, ask them
   whether the step is part of it, with or without a selected Automation
   revision, and go on only on a clear yes. On a team without automation
   revisions, a save goes into the draft you took and into no Automation
   draft. A `selected_automation_revision_id` exists only on a team with
   automation revisions, so with one, ask about the selected draft as above.
   With none, when you cannot tell which kind of team it is, ask one question
   that covers both: this draft may hold an outdated version of the step, and if
   the step's Automation uses revisions (it has an active revision or a
   draft), the save also puts the step into its open draft, creating that
   draft if there is none. Is this the version they want to edit, and, if the
   Automation uses revisions, should the step be in that draft? Go on only
   on a clear yes to both; when the user says the Automation does not use
   revisions, a yes to the first is enough.
   With a `selected_automation_revision_id`, the draft you took may belong to
   another Automation revision. Once the user has said yes, the selected
   Automation draft holds a draft of the step, and the server saves to that
   draft rather than to the one you took: the one you took only supplies the
   hash, so the save either changes nothing or is refused with the right
   draft's `build_id`. Confirm the draft the same way as in 1: send it its own
   `config` unchanged, with its `config_hash` and `require_draft: true`. The
   revision id it returns, or the `build_id` of a `revision_config_changed`
   refusal, is the one to edit. Without a selected id, use the draft you took.

Then read the revision to edit (section 3) and build your change on it.

Take a draft from the listing only after the save to the live revision is
refused with `revision_not_draft`, or when the step has no live revision; in
that second case, ask and confirm as 2 says. `listAgentRevisions` does not
know which Automation revision is selected, and it can list a draft left
behind by an Automation draft that was discarded. When the target Automation
draft (the selected one, or the Automation's open draft) has not changed the
step yet, the server saves to the draft you name as it is, bringing the
discarded change back.

From then on, the revision id a save returns, or the `build_id` a
`revision_config_changed` refusal names, is the draft to read and save, under
the same `selected_automation_revision_id`. When it changes, read its config
and files from scratch (section 3) before you build the change again. Never
reuse content, a hash or a file list you read from another revision.

## 3. Read what is there now

1. `getRevision` on the revision from step 2. Keep its whole `config` and its
   `config_hash`.
2. `listCodeStepFiles` on the same revision. It returns a page at a time: read pages
   until you hold `total` files before you decide a file is missing.
3. `getCodeStepFile` (by `path`) for each file you will change, and for any
   file you need in order to understand the change. If chat refuses a file as
   over its character limit, do not retry it:
   - For an inline step, `main.py` is already in hand: it is `source.code` in
     the config you read with `getRevision`. Work from that.
   - For a multi-file step there is no smaller read. Tell the user that file
     is too large to edit here and to change it in the builder.

**A result cut short is not the file.** A tool result over about 20 KB comes
back shortened, with its middle left out: it says the output was truncated
(`Warning: truncated output` on the harness) and names a file that holds the
whole response. This happens to ordinary files, well below the size chat
refuses. Never build, upload or check a file from the shortened text. The saved
file holds the response as JSON on a single line, too long for the read tool:
either the response itself, or a list of text blocks that carry it as a string.
Take the content out with the shell and work from that:

```bash
unwrap='if type == "array" then map(.text // empty) | join("") | fromjson else . end'
jq -je "$unwrap | .file.content | strings" <saved file> > helpers.py                # getCodeStepFile
jq -je "$unwrap | .build.config.data.source.code | strings" <saved file> > main.py  # getRevision, inline step
```

`-j` writes the text exactly, with no added newline. `-e` with `strings` makes
the command fail, instead of writing an empty file, when the field is not
there: stop if it fails. Then read the written file in parts if it is long.

`config.data.source` says how the program is stored:

- `kind: "inline"`: one file, `main.py`, held in `source.code`.
- `kind: "files"`: a manifest in `source.bundle`, with `bucket` and `files`,
  each file `{ path, sha256, sizeBytes }`, and `startCommand`, the command it
  runs (required; usually `python main.py`).

## 4. Build the new config

Change only `config.data.source`. Keep every other field of the config you
read exactly as it is.

- **Change the program of an inline step:** put the new program in
  `source.code`. It must stay under 128 KB.
- **Change one file of a multi-file step:** `uploadCodeStepFile` with the
  step's `agent_id` and the file's full new content. Replace that file's entry
  with its `path` and the `sha256` and `sizeBytes` the upload returned. Leave
  the other entries as they are.
- **Add a file to a multi-file step:** upload it and add its entry.

There is one file store, so the `bucket` an upload returns is the bundle's
`bucket`. If it is not, the files already in the bundle cannot be read or kept:
stop and tell the user, and do not save.

- **Add a second file to an inline step:** write the new `main.py` (start
  from `source.code` and apply the change, such as importing a helper you move
  out and deleting it from `main.py`), upload it and the new file, then
  replace the source with
  `{ kind: "files", startCommand: "python main.py", bundle: { bucket, files } }`
  holding both entries. Every file you upload is its complete new content.
- **Remove a file:** drop its entry. Uploading alone changes nothing; a file is
  part of the program only once its entry is saved.

Keep the entry file. A program run with `python main.py` must keep `main.py`. A
program with its own `startCommand` must keep the file that command names;
nothing checks that for you. A program holds at most 200 files, each up to
128 KB, 4 MB in total.

## 5. Save with the read's hash

Call `updateRevision` on the revision you read with the whole new `config`,
`expected_config_hash` set to the `config_hash` you read in step 3, and
`require_draft: true`. Send no `handoverTargetIds`; a code step has none, and
leaving them out keeps the stored ones.

The hash is what keeps your save from erasing someone else's. The builder may
be open on the same draft and autosaving, or another chat may have saved it.

- **409 with code `revision_config_changed`:** the revision the save landed on
  changed after your read, or already held other changes, and nothing was
  saved. Its message names the revision the save landed
  on (`build_id`); use that id from now on, since rereading a different draft
  would be refused again. Keep the same `selected_automation_revision_id`. Go
  back to step 3: read that revision's config, file list and every file you
  change or need, apply your change to what it holds now, upload again any
  file whose content differs, and save with its new `config_hash`. If the
  other change touched the same file, tell the user what changed and check the
  change still makes sense before you save. Never retry the old config, and
  never drop `expected_config_hash` to get past the refusal.
- **409 with code `revision_not_draft`:** the save would have changed a
  revision that is not a draft, so it was refused. On the first save to the
  live revision, step 2 says what this means. Later, the draft was made live
  after your read: go back to step 2 and start again from the live revision,
  under the same `selected_automation_revision_id`, reading everything again.
- **409 with code `revision_generation_pending`:** a generation on the draft
  is running or waiting for review in the builder, and discarding it would
  erase your save. Tell the user to keep or discard it there first, then try
  again.
- **Any other refusal:** quote it to the user. Do not work around it.

## 6. Check and show the result

1. `listCodeStepFiles` on the revision the save returned: every file you meant
   is there, and the entry file is still there.
2. `getCodeStepFile` for each changed file: the content is what you wrote.
   For an inline step whose `main.py` chat refuses as too large, read
   `source.code` with `getRevision` on the saved revision instead. Compare
   against the whole file: take a result cut short from its saved file as in
   section 3, into a new file (such as `saved-main.py`) so the copy you edited
   stays as it is, then `diff` the two.
3. Show the user the changed lines of each file, not the whole program, and
   name the step and the draft revision.

Do not start a Run to test the draft: `startRun` runs the live revision, not
the draft. If the user wants to try the change first, tell them to open the
draft in the builder and run it there: a Run started with the draft selected
runs the draft.

## 7. Going live is the user's call

The draft is the review. Triggers and Runs started outside the builder
keep using the live revision until the draft is made live; only a Run
the user starts in the builder with the draft selected runs it. Say "saved to
a draft; it is not live yet".

- **The step is in an Automation with revisions** (the save landed on the
  Automation's draft): the user makes that draft active on the Automation's
  page, together with everything else in it. Tell them so; do not call
  `promoteRevision` for it.
- **Otherwise:** make it live with `promoteRevision` only after the user asks
  for it and confirms it the way your host's rules for an activation say. In
  Ask Duvo that is the `Activate` question card: it quotes the step's name and
  the revision's name exactly as they are named, offers `Activate` and
  `Keep as draft`, and carries `activate:<revision id>` where the chat's rules
  put it (the card's `metadata.source` in the classic chat, the question's
  `id` with `askUserQuestions`). Promote only after the user picks `Activate`.

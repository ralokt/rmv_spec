
# RMV specification

THIS IS STILL A DRAFT: suggestions for improvements before release are welcome.

This is the specification for the RMV file format.

"I" in this document refers to Thomas Kolar aka ralokt, but the format was
originally designed by Christoph Nikolaus. It has since been expanded upon by
me and Elias Gailberger aka KharadBanar.

Part of the design decision section is opinionated, but the spec section aims
not to be.

## Changes in RMV v2

RMV v2 is a backwards-incompatible update of the format.
 - removed utf-8 property. All text data is now always utf-8.
 - removed text-based result string.
 - removed timestampchange event type.
 - added bbbv_low and bbbv_high properties (low and high bits of bbbv).
 - added square_size property.
 - mouse coordinates are now signed, and relative to the top left corner of the
   board, as opposed to the top left corner of the UI.
 - added reduced mouse move event that
   - uses relative coordinates/timestamps
   - omits nFlags
   - and therefore fits into 3 bytes instead of 9
 - added machine-readable fields to specify the clone used, along with its
   major version.
 - added named extension properties. Clones that want to add metadata that is
   not part of the specification may put it here, with the goal that if they
   are found useful, they are added to the format in later versions.
 - added EVF game modes

## Authorship and future changes

Responsibility for making changes to this format rests with `ralokt`, or
`KharadBanar` should `ralokt` get hit by a bus or otherwise be dead or
incapacitated. From now on, that person is referred to as the RIDAA (RMV ID
assignment authority), and both (if available) together form the RSC (RMV
specification committee).

Note: We're not actually formally founding an organization, we just want to use
the abbreviations above when discussing how changes to the file format will be
handled.

### How to contact

#### ralokt

 - E-Mail: thomaskolar90@gmail.com
 - Discord: @ralokt

#### KharadBanar

 - E-Mail: kharadbanar@gmail.com

## Design goals/decisions, comparison to alternatives

First and foremost, by "alternatives", I mean EVF.

MVR is not an open format, and if you're considering AVF, do yourself a favor
and reconsider. In particular, while RMV makes some different tradeoffs/design
decisions, EVF is superior to AVF in pretty much every way, and there is no
reason to use AVF over it.

The EVF spec can be found here:

https://github.com/eee555/ms-toollib/blob/main/evf%20format%20(evf%E6%A0%87%E5%87%86).md

### Design goals

The original design goals aren't quite known, as Christoph Nikolaus, the
original author of the format, has resigned. However, we can infer the
following:
 - RMV is meant to be a universal format (currently, it's only being used by
   vsweep)
 - RMV is meant to be usable for the official world ranking (and it is)
 - RMV prioritizes ease of use over file size (it stores board events alongside
   mouse events - see below for details)

### Design decisions

#### Board events

Unlike AVF or EVF, RMV also contains *board events* - events that describe how
the state of the game changed in response to the the previous mouse event.

This approach has a few advantages:
 - It makes writing replay viewers extremely simple. All other formats require
   implementing a Minesweeper engine.
 - It documents a game as it was played, even if the implementation was buggy.
   For example, an implementation with a bug like rilian clicks would still
   produce replays that show the game as it was played, not as it *should* have
   been played if the implementation hadn't been buggy.

   Of course, with formats that don't store board events, it's still possible to
   parse the version info to account for bugs in specific versions; but now
   developers don't just have to implement a Minesweeper engine, but also make
   it conditionally support buggy behavior.
 - Well-behaved clones implementing RMV are essentially factories for test data
   for new Minesweeper engines, as they document the correct result for every
   action in every game that is played and saved on them.

It also has some disadvantages:
 - File size is increased. While by far the most events in Minesweeper replays
   are mouse move events, it's not insignificant, either.
 - While there are upsides to this, as outlined above, the format contains
   redundant information.

EVF does not contain board events - this is the greatest conceptual difference
between the two formats.

### Deviations from EVF

#### No UUID field

EVF supports a UUID field that may contain a device UUID for the device the game
was played on.

This feature isn't a reliable measure against fake replays, as the UUID is by
definition stored in any videos a user publishes, and can be spoofed.

For less savvy users, the UUID links games to a specific device. That includes,
say, games played on a company computer during a meeting that could have been an
email, and has privacy implications in general.

Besides, asymmetric encryption would be a better solution for the problem the
UUID field aims to solve.

It would be great if EVF and RMV converged more in general, but this is why RMV
makes an explicit decision not to implement that.

#### Always utf-8 starting in RMV version 2

Starting with RMV version 2, all text data is explicitly specified to contain
utf-8. Parsers can and should reject replays that don't fulfil this requirement.

This makes writing (proper) parsers easier. It also means that for any
translated replays, any text data needs to be converted.

## Specification

### Format

#### How to read the structure section

The structure is given as a list. Each list item contains one of the following:
 - A definition consisting of a value, followed by "->", followed by a name,
   where values are one of:
   - `chars(<string_literal>)` Just these characters. For example:
     `chars("asdf")`.
   - `uint(<length>)` - an unsigned integer encoded in `length` bytes.
   - `sint(<length>)` - a signed integer encoded in `length` bytes.
   - `sint(<length>bits)` - a signed integer encoded in `length` bits.
   - `str(<length>)` - a text string encoded in `length` bytes, where `length`
     is given as the name of a previously parsed result. If the `utf8` property
     is set or `format_version >= 2`, this contains utf-8. See below for
     handling utf-8 fields.
   - `blob(<length>)` - a binary blob that is `length` bytes long, where
     `length` is given as the name of a previously parsed result
   - an expression using previously defined values
 - `if <condition>` - makes following indented list conditional
 - `repeat <amount> times` - `<amount>`-fold repetition of the following
   indented list, where `amount` is given as the name of a previously parsed
   result
 - `repeat until <name> defined` - repetition of the following indented list
   until a value is defined as `name`

Conditions, string literals, and expressions are loosely defined as pseudocode
in the interest of sacrificing theoretical precision for readability.

In the interest of keeping the format definition compact, its structure will be
defined first, and semantics of each value will get their own paragraph later.

##### Handling utf-8

Clones MUST emit valid utf-8 to utf-8 fields.

Reader implementations MUST either
- completely ignore the contents of these fields
- OR verify that they are valid utf-8
- OR:
  - treat the data as an opaque binary blob, performing no processing that
    implicitly assumes an encoding
  - AND be written in a programming language/environment in which there are no
    facilities readily available to perform the validation/conversion
    - (ie: this shouldn't be a blocker to writing a parser in ASM or brainfuck,
      but if your standard library allows you to do it, then effing do it)

Readers SHOULD decode utf-8 fields and expose them as text data. Readers MAY
instead expose utf-8 fields 1:1 as binary blobs (but, as stated above, MUST
still verify that they contain valid utf-8 in that case).

Every complaint about this part of the spec MUST be accompanied by a
demonstration that the complaining party has read and understood the following
blog post from 2003:

https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/

To demonstrate that you have read the blog post, please include:
 - what the single most important fact about encodings is
 - what you will spend the next 6 months doing in a submarine, and why you
   think that that doesn't apply to you

Otherwise, your complaint may be disregarded.

(Notes:
 - Since ASCII is a subset of utf-8, clones can emit that and still be
   compliant
 - If your existing library doesn't parse text data, all you have to add is a
   validity check - that's what the third paragraph is for. If it does parse
   text data, but in a broken way, then fix it.)

#### Structure

 - `"*rmv"` -> `file_signature`
 - `uint(2)` -> `file_type`
 - `if file_type == 2`:
   - `uint(1)` -> `clone_id`
 - `if file_type == 2`:
   - `uint(1)` -> `major_version_of_clone`
 - `uint(4)` -> `file_size`
 - `if file_type == 1`:
   - `uint(2)` -> `result_str_size`
 - `uint(2)` -> `version_info_size`
 - `uint(2)` -> `player_info_size`
 - `uint(2)` -> `board_size`
 - `uint(2)` -> `preflagged_size`
 - `uint(2)` -> `properties_size`
 - `if file_type == 2`:
   - `uint(2)` -> `extension_properties_size`
 - `uint(4)` -> `vid_size`
 - `uint(2)` -> `checksum_size`
 - `if file_type == 1`:
   - `str(result_str_size)` -> `result_str`
 - `str(version_info_size)` -> `version_info`
 - `uint(2)` -> `num_player_fields`
 - `repeat num_player_fields times`
   - `uint(1)` -> `player_field_size`
   - `str(player_field_size)` -> `player_field`
 - `uint(4)` -> `timestamp_boardgen`
 - `uint(1)` -> `cols`
 - `uint(1)` -> `rows`
 - `uint(2)` -> `num_mines`
 - `repeat num_mines times`
   - `uint(1)` -> `col`
   - `uint(1)` -> `row`
 - `if preflagged_size != 0`:
   - `uint(2)` -> `num_preflags`
   - `repeat num_preflags times`
     - `uint(1)` -> `col`
     - `uint(1)` -> `row`
 - `repeat properties_size times`
   - `uint(1)` -> `property`
 - `if file_type == 2`
   - `uint(2)` -> `num_extension_properties`
   - `repeat num_extension_properties times`
     - `uint(1)` -> `extension_property_name_size`
     - `str(extension_property_name_size)` -> `extension_property_name`
     - `uint(1)` -> `extension_property_value_size`
     - `blob(extension_property_value_size)` -> `extension_property_value`
 - `repeat until time_ms defined`
   - `uint(1)` -> `event_code`
   - `if file_type == 1 && event_code == 0`
     - `event_code` -> `timestampchange_event_type`
     - `uint(4)` -> `timestamp`
   - `if 1 <= event_code <= 7`
     - `event_code` -> `mouse_event_type`
     - `uint(3)` -> `gametime`
     - `uint(1)` -> `nFlags`
     - `if file_type == 1`
       - `uint(2)` -> `xpos`
       - `uint(2)` -> `ypos`
     - `else`
       - `sint(2)` -> `xpos`
       - `sint(2)` -> `ypos`
   - `if 9 <= event_code <= 14 or 18 <= event_code <= 27`
     - `event_code` -> `board_event_type`
     - `uint(1)` -> `col`
     - `uint(1)` -> `row`
   - `if 15 <= event_code <= 17`
     - `event_code` -> `termination_event_type`
     - `uint(3)` -> `time_ms`
   - `if file_type == 2 and event_code == 28`
     - `event_code` -> `mouse_event_type`
     - `uint(1)` -> `gametime_change`
     - `sint(4bits)` -> `xpos_change`
     - `sint(4bits)` -> `ypos_change`
 - `blob(checksum_size)` -> `checksum`

#### Semantics

##### `file_signature`

This is just the file signature/magic number for RMV.

##### `file_type`

The format version.

Allowed values:
 - `1` (original RMV format)
 - `2` (RMV2 format)

This will be changed when the file format receives a backwards-incompatible
update.

##### `clone_id`

A unique ID for the clone that produced this replay, assigned by the RIDAA.

Clones that don't have an assigned value yet MUST set this value to `0` and MUST
use the `clone_name` extension property instead (see below).

Clones that have an assigned value MUST use that value, and MUST NOT set the
`clone_name` extension property.

This value is used to disambiguate extension properties. Since clones can
define their own extension properties, collisions can be resolved using this
field. It also serves as a machine-readable field for the clone that produced
the replay.

Fixed values:
 - `0` (unspecified)
 - `1` (vsweep)
 - Any other value, may have been assigned later (degrade to parsing
   universal data)

The following values are assigned proactively, but may still be reassigned. If
newer versions of these clones want to write RMV replays, they can use these
values right away, skipping the step of using the `clone_name` extension
property. They should inform the RIDAA that they are claiming their preassigned
ID, fixing it forever.
 - `2` (arbiter)
 - `3` (MSX)
 - `4` (clone)
 - `5` (metasweeper)
 - `6` (freesweeper)
 - `7` (mysweeper)

If the RIDAA ever runs out of IDs, they may reassign an unclaimed ID, should the
developers of the respective clones not be reachable/responsive.

Clone authors may request an ID from the RIDAA. Depending on demand, the RIDAA
may ask for proof that a project is being seriously worked on/in an advanced
stage.

In order to be eligible, clones should aim to be suitable for use in the
official world ranking on minesweepergame.com . This is a loose requirement:
 - Clones don't need to be usable on the world ranking yet
 - Clones don't even need to have all of the necessary features yet
However, clones should not be fundamentally incompatible with being used on the
world ranking.

The RIDAA (current and future) shall not cite any considerations besides the
above as reasons for not assigning an ID to a project. They exist as a mechanism
to prevent different projects from claiming the same ID, and to ensure that the
limited space of 255 IDs isn't exhausted without reason. They do NOT exist as
gatekeepers, and their goal shall be to get projects an ID assigned as soon as
possible, regardless of any personal considerations.

Fixed IDs will never be reassigned. Instead, if necessary, a new version of the
format will be released.

##### `major_version_of_clone`

The major version of the clone that generated the replay.

It's recommended to increment this field whenever parsers could benefit from
telling the difference between this version and the previous version - for
example, when new extension properties are assigned, or when a bug is fixed/a
feature is added that affects replay generation.

It's recommended to release new major versions whenever the last paragraph
applies, or to make the contents of this field align with such changes instead
of the actual major version number.

##### `file_size`

Size of the entire file, in bytes.

##### `result_str_size`

Size of `result_str`, in bytes. Only present in version 1 of the format.

##### `version_info_size`

Size of `version_info`, in bytes.

##### `player_info_size`

Size of `player_info`, in bytes.

##### `board_size`

Size of `board` section, in bytes. The `board` section contains:
 - `timestamp_boardgen`
 - `cols`
 - `rows`
 - `num_mines`
 - The `col, row` pairs that define the positions of the mines.

See below for details.

##### `preflagged_size`

Size of `preflagged` section, in bytes. The `preflagged` section contains:
 - `num_mines`
 - The `col, row` pairs that define the positions of the mines.

See below for details.

##### `properties_size`

Size of `properties` section, in bytes. Also equal to the number of properties,
as properties are 1 byte each.

See below for details.

##### `extension_properties_size`

Size of `extension_properties` section, in bytes. Only present in version 2 of
the format.

See below for details.

##### `vid_size`

Size of the `vid` section, in bytes. The `vid` section contains the events.

##### `checksum_size`

Size of `checksum`, in bytes.

See below for details.

##### `result_str`

Result string. Contains metadata about the game in text format.

Only present in version 1 of the format, so it will only be documented briefly
and informally.

Contains `"\n" (<key> ":" <value> "#")* "\n"`. The keys that may be available
are:
 - `LEVEL`
 - `SCORE`
 - `NAME`
 - `NICK`
 - `3BV`
 - `NF`
 - `TIMESTAMP`

Of these, only `3BV` is interesting for modern parsers, as it isn't available
elsewhere in the format in version 1.

##### `version_info`

A free-form human-readable version information string.

##### `num_player_fields`

The number of player fields. See below.

##### `player_field_size`

Size of the next player field, in bytes.

##### `player_field`

Contents of the next player field.

Player fields are positional.

The player fields are:
 - `name`: Intended for the player's full/legal name
 - `nickname`: Intended for the player's nickname/online handle/however they
   want to be addressed in an online context.
 - `country`: The player's country as a free-form text field.
 - `token`: A secret that proves that the game was played by a player knowing
   that secret. For example, a Scoreganizer tournament key.

Fields are assumed to be empty if not present.

If you are implementing RMV, do not add your own player fields; instead, add an
extension property (see below).

##### `timestamp_boardgen`

The timestamp at which the board was generated.

##### `cols`, `rows`, `num_mines`

The number of columns, rows, and mines for the board the game was played on.

##### `col`, `row` (in board section)

The column and row the next mine was placed in.

##### `num_preflags`

How many flags were placed before the game started.

##### `col`, `row` (in preflags section)

The column and row the next preflag was placed in.

##### `property`

Properties are positional. In both format versions, the first 4 properties are:
 - `marks` - whether questionmarks were enabled
 - `nf` - whether the game was a non-flagging game
 - `mode` - the game mode:
   - `0` - normal
   - `1` - upk
   - `2` - cheat
   - `3` - density
   - `4` - win7
   - `5` - competitive_solvable
   - `6` - strong_solvable
   - `7` - weak_solvable
   - `8` - to_be_solvable
   - `9` - strong_guessable
   - `10` - weak_guessable
   - `11` - chording_recursive_standard
   - `12` - flag_recursive
   - `13` - chording_flag_recursive
 - `level` - the level played:
   - `0` - beginner
   - `1` - intermediate
   - `2` - expert
   - `3` - custom

Values for `mode` that are larger than 3 are only valid in version 2 of the
format.

In version 1, the following extra property may exist:
 - `utf8` - whether or not text strings in the replay file are encoded as utf-8.
   if the property doesn't exist, the encoding isn't defined. (In version 2, the
   encoding must always be utf-8, and this property no longer exists.)

In version 2, the following extra properties exist:
 - `bbbv_low` - low 8 bits of the 3bv of the board
 - `bbbv_high` - high 8 bits of the 3bv of the board
 - `square_size` - side length of squares the game was played on, in pixels

If you are implementing RMV, do not add your own player fields; instead, add an
extension property (see below).

##### `num_extension_properties`

The number of extension properties. Only present in version 2 of the format.

Extension properties are discussed in their own section below.

##### `extension_property_name_size`

Size of the following extension property name, in bytes. Only present in version 2 of the format.

Extension properties are discussed in their own section below.

##### `extension_property_name`

The next extension property name. Only present in version 2 of the format.

Extension properties are discussed in their own section below.

##### `extension_property_value_size`

Size of the following extension property value, in bytes. Only present in version 2 of the format.

Extension properties are discussed in their own section below.

##### `extension_property_value`

The next extension property value. Only present in version 2 of the format.

Extension properties are discussed in their own section below.

##### `event_code`

Determines the type of the next event. The exact meanings are described in the
`*_event_type` sections below.

##### `timestampchange_event_type`

Only valid in version 1 of the format, and only used in very old replays. Always
`0` if present.

##### `timestamp`

New current timestamp. Only present in version 1 of the format.

##### `mouse_event_type`

Mouse events - moves, button presses, and button releases.

Allowed values and meanings:
 - `1` - mouse move
 - `2` - lmb_down
 - `3` - lmb_up
 - `4` - rmb_down
 - `5` - rmb_up
 - `6` - mmb_down
 - `7` - mmb_up
 - `28` - reduced mouse move

SHOULD simply correspond 1:1 to mouse events that happened ingame. MUST do so if
the middle mouse button was not used.

Reduced mouse move SHOULD be written by clones when:
 - `nFlags` didn't change compared to the last event
 - the mouse position change in both axes is between -8 and 7 inclusive
 - the current gametime is 255ms or less after the last event

If one of these conditions is not met, clones MUST write a classical mouse move

It implies a mouse move event with:
 - the same value of `nFlags` as the previous event
 - a `gametime` that is offset from the gametime of the previous event
   by `gametime_change` milliseconds
 - an `xpos` that is offset from the `xpos` of the previous event
   by `xpos_change` pixels
 - a `ypos` that is offset from the `ypos` of the previous event
   by `ypos_change` pixels

##### `gametime`

Time since start of the game, in milliseconds.

##### `nFlags`

Bitflags describing which mouse buttons were pressed.
TODO: describe in detail.

##### `xpos`, `ypos`

x and y positions of the cursor at the time of the event, in pixels.

In format version 1, both are unsigned, and relative to the top left corner of
ViennaSweeper's client area. `xpos` is offset by `12`, while `ypos` is offset by
`56`.

In format version 2, both are signed, and relative to the top left corner of the
board.

##### `board_event_type`

Board events - a square changing what it displays.

Allowed values and meanings:
 - `9` - pressed
 - `10` - pressed questionmark
 - `11` - closed
 - `12` - qm
 - `13` - flag
 - `14` - open (works as open_blast)
 - `18` - open_0
 - `19` - open_1
 - `20` - open_2
 - `21` - open_3
 - `22` - open_4
 - `23` - open_5
 - `24` - open_6
 - `25` - open_7
 - `26` - open_8
 - `27` - open_blast

##### `col`, `row`

Board coordinates of the square that was affected.

##### `termination_event_type`

Termination events - represent the game ending.

Allowed values and meanings:
 - `15` - blast
 - `16` - win
 - `17` - other - when the game ended in neither a win or a blast - system
   crash, network failure, accidentally clicking the smiley, ...

##### `time_ms`

Gametime, in thousandths, since the start of the game. Technically part of the
termination event, and included in `vid_size`. However, since it marks the end
of the game, can be seen as a standalone field that contains the total game
time.

##### `checksum`

A checksum that is calculated based on the rest of the replay file. Should not
contain information from:
 - `file_signature`
 - `file_type`
 - `clone_id`
 - `major_version_of_clone`
 - `file_size`

#### Extension properties

Properties and player fields are positional. This means that if clone authors
decided to extend the format independently, a situation could arise where
different clones interpret different properties/player fields differently.

Therefore, clone authors that are not members of the RSC MUST NOT define new
properties. However, they can define *extension properties*.

Unlike properties and player fields, extension properties are named - they
contain a name as well as a value. They are also intended to be interpreted in
tandem with the `clone_id`, as opposed to properties and player fields, which
are intended to be universal. In other words: extension properties being named
makes collisions less likely, and `clone_id` provides a fallback.

Extension properties are defined as blobs. This is so that they can contain
actual binary data, for example, integer values. However, if they contain text
data, it MUST be encoded in utf-8.

##### Reserved extension properties

The following extension properties are reserved:
 - `clone_name` - MUST NOT be set by clones that have an assigned `clone_id`.
   MUST be set by clones that don't have an assigned `clone_id`.
 - All properties starting with `rmv_`

##### Suggested conventions

We suggest that clone authors use the following conventions when defining
extension properties:
 - In general: follow the convention `lowercase_with_underscores`
 - For statistics: prefix with `stat_`. If the statistic is an abbreviation, use
   the lowercase version of that, otherwise, use the full name.
   - Examples: `stat_islands`, `stat_openings`, `stat_stnb`, `stat_hzini`
 - For fields that relate to the player and could become player fields in the
   future: prefix with `player_`.
   - Examples: `player_signature`, `player_saolei_id`, `player_mouse_model`
 - For fields that could become a universal property: prefix with `prop_`
   - Examples: `prop_show_arena_ticket_drops`, `prop_rilian_clicks`,
     `prop_openings_destroy_flags`, `prop_elmar_technique`
 - For fields that are specific to your clone, prefix with `<clone_name>_`.
   - Examples: `vsweep_skin`, `msx_skin`, `arbiter_clipcursor`,
     `metasweeper_uuid`

#### Future additions to properties and player fields

More properties and player fields may be added to version 2 of the format by the
RSC, if doing so is possible without breaking backwards compatibility.

This means:
 - Parsers should read the properties they understand and ignore the rest
 - Updates to the format must not render the previous point problematic

An example for a backwards compatible change is the new `token` player field
that was added with ViennaSweeper 4.0.0. It contains extra information that can
be used to perform additional checks, but can be safely ignored to simply read a
replay file.

Backwards compatibility doesn't have to be perfect. For example, a RMV file with
the `utf8` property would still have been interpreted by older versions of
ViennaSweeper as containing strings in `latin1`, or some other native code page,
leading to some display errors. However, the RMV file would still have been
playable.

An example for a backwards-incompatible change to RMV is the new `square_size`
property. This is not backwards compatible, as unless `square_size` is set to
16, replays in the new format would not be playable by a player that doesn't
understand this property - the mouse position relative to the board position
where changes happen would be incorrect.

In fact, `square_size` being necessary to support arbitrary square sizes in
ViennaSweeper is the reason why RMV v2 is necessary.

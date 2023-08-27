**Introduction**

**tags2sqlite**:  is a collection of Python scripts to import audio metadata from underlying audio files into a dynamically created SQLite database, allowing you to affect mass updates / changes using SQL and ultimately write those changes back to the underlying files.  It leverages the puddletag codebase so you need to either install puddletag or at least pull it from the git repo to be able to access its code, specifically puddletag/puddlestuff/audioinfo.  Tags are read/written using the Mutagen library as used in Puddletag. Requires Python 3.x

General philosophy and rationale:
I have a relatively large music collection and I rely on good metadata to enhance my abilility to explore my music collection in useful and interesting ways.  Taggers are great but can only take you so far.  Tag sources also vary in consistency and quality, often times including things like ```feat artist``` in track tags, making it more difficult for a music server to identify the track as a composition and thus include the particular performance in a list of performances of said composition.  tags2sqlite lets you automatically make pre-coded changes to these sorts of issues and does a lot of cleanup work that is difficult to do within a tagger.  It also does it at scale, repeatably and consistently, no matter whether you're handling 1,000 or 1,000,000 tracks - this is simply an impossible task prone to variation and human error when done manually via a tagger.  It is somewhat biased toward embedding ```MusicBrainz MBID's``` where available as music servers are increasingly leveraging the ```MusicBrainz MBID's``` when present. It writes its changes to a database table and keeps track of which tracks have had metadata changes.  It does not make changes to your files.  All tables in the database can be browsed using a SQLite database editor like ```Sqlitestudio``` or ```DB Browser for SQLite``` so you can inspect tags and see exactly what would be written to files if you chose to export your changes back to the underlying files.  After running the automated changes you're also able to manually edit any records using the aforementioned database editors to further correct things manually or through SQL queries if you're so inclined.  Finally, all originally ingested records are written to a rollback table, so in the event you've made changes you don't like you can simply reinstate your old tags by exporting from the rollback table.

If your library is static in terms of file location one could also use the metadata database as a means of backing up and versioning metadata simply by keeping various iterations of the database, obviating the need to overwrite a previous backup of the underlying music files.  Getting metadata current after restoring a dataed backup of your music files would be as simple as exporting the most recent database against the restored files.  The added benefit is no need to create incremental backups of your music files simply because you've augmented the metadata - just backup the datavase and as long as your file locations remain static you have everythig you need.


Understanding the scripts:
**tagfromdb3.py** handles the import and export from/to the underlying files and SQLite database.  Basically it is the means of getting your tags in and out of your underlying audio files.  This is where the puddletag depedency originates.  I've modded Keith's Python 2.x code to run under Python 3.  To get it to work all that's required is that you pull a copy of puddletag source from github: https://github.com/puddletag/puddletag, then copy tagfromdb3.py into the puddletag root folder.  You do not need a functioning puddletag to be able to use tagfromdb3.py, albeit in time you might find puddletag handy for some cleansing/ editing that's best left to human intervention.

**dropbannedtags.py** does the heavy lifting handling the cleanup of tags in the SQL table "alib".  A trigger is used to be able to isolate and write back tags only to files who's tag record has been modified (the trigger field sqlmodded is incremented every time the record is updated)
At present it does the following:

General tag cleanup:
- strips all spurious tags from the database so that your files only contain the sanctioned tags listed in ```dropbannedtags.py``` (you can obviously modify to suit your needs).
- trims all text fields to remove leading and trailing spaces
- removes all spurious ```CR/LF``` occurences in text tags.  It does not process the ```LYRICS``` or ```REVIEW``` tags
- removes ```PERFORMER``` tags where they match or are already present in the ```ARTIST``` tag
- eliminates duplicate entries in tags

Tag standardisation:
- merges ```ALBUM``` and ```VERSION``` tags into ```ALBUM``` tag to get around Logitechmediaserver (LMS), Navidrome and other music servers merging different versions of an album into a single album.  ```VERSION``` is left intact making it simple to reverse with an UPDATE query
- sets ```COMPILATION``` = ```1``` for all Various Artists albums and to '0' for all others.  Tests for presence or otherwise of ```ALBUMARTIST``` and whether ```__dirname``` of album begins with ```VA - ``` to make its deterimation
- removes 'Various Artists' as ```ALBUMARTIST```
- writes out multiple ```TAGNAME=value``` rather than ```TAGNAME=value1\\value2``` delimited tag entries
- Normalises ```RELEASETYPE``` entries for using First Letter Caps for better presentation in music server front-ends that leverage it
- adds musicbrainz identifiers to artists & albumartists leveraging what already exists in file tags.  Where a performer name is associated with > 1 mbid in your tags these performers are ignored so as not to conflate performers.  Check tables: namesakes_* for contributors requiring disambiguation

Handling of ```Live``` in album names and track titles:
- removes all instances and variations of ```Live``` entries from track titles and moves or appends that to the ```SUBTITLE``` tag as appropriate and ensures that the ```LIVE``` tag is set to ```1```  where this is not already the case
- removes (live) from end of all album names, sets ```LIVE``` = '1' where it's not already set to '1' and appends (Live) to ```SUBTITLE``` tag where this is not already the case
- ensures ```LIVE``` tag is set to ```1``` for all Live performances where ```[(live...)]``` appears in ```TITLE``` or ```SUBTITLE``` tags

Handling of 'Feat' in track title and artist tags:
- removes all instances and variations of ```Feat``` entries ```ARTIST``` and ```TITLE``` tags and appends the delimited performer names to the ```ARTIST``` tag

Identifying duplicated audio content:
- identifies all duplicated albums based on records in the alib table.  The code relies on the md5sum embedded in properly encoded FLAC files - it basically takes them, creates a concatenated string
    from the sorted md5sum of all tracks in a folder and compares that against the same for all other folders.  If the strings match you have a 100% match of the audio stream and thus duplicate album, irrespective of tags / metadata.  You can condifently remove all but one of the matched folders.


At present must be started in root of tree you intend to import.
I strongly suggest writing the SQLite database to ```/tmp``` as its alib table is dynamically modified every time a new tag is encounted when the taga from a file are being imported.  It'll work on physical disk, but it'll take longer.  It'll also trigger a lot of writes whilst ingesting metadata and dynamically altering the table to ingest new tags, so you probably want to avoid hammering a SSD by esuring that you're not having the database written directly to a SSD drive.  Use ```/tmp```!

TODO:

- incorporate metadata normalisation routines to standardise case of track ```TITLE```, ```PERFORMER```, ```COMPOSER``` & ```LABEL``` metadata
- leverage cosine similarity to generate potential duplicate in contributor metadata requiring manual intervention 
- add Musicbrainz identifiers to all ```ARTIST```, ```PERFORMER```, ```COMPOSER```, ```LYRICIST```, ```WRITER```, ```LABEL```, ```WORK```, ```PART``` and ```ALBUMARTIST``` tags
- consider adding musicbrainz_composerid for future app use
- remember to search for ```ARTIST``` and ```ALBUMARTIST``` with ```\\``` where ```musicbrainz_artistid``` and ```musicbrainz_albumartistid``` not like ```\\``` to ensure additional mbid's are added where appropriate.
- incorporate metadata enrichment leveraging Musicbrainz and inferences based on existing track related metadata in table
- cleanup and standardise genres to eliminate unsanctioned ```GENRE``` entries
- enrich "Pop/Rock", "Jazz" & "Classical" only genre assignments with artist based ```GENRE``` and ```STYLE``` entries
- ensure completeness of various tags across all tracks in a folder/album e.g. all tracks have ```DATE``` and ```GENRE``` assignments and that they're the same (albeit some users will not want track genres homoginised for an album), so keept the code seperate
- merge ```GENRE``` and ```STYLE``` tags to ```GENRE``` tag and dedupe
- write out __dirpaths for various queries to additonal tables users can use to focus on manual adjustments e.g. adding ```DATE``` tag to albums without dates, ```GENRE``` tag to albums without genres etc.
- fill in the blanks on ```ALBUMARTIST```, ```GENRE```, ```STYLE```, ```MOOD``` & ```THEME``` tags
- implement argv to enable passing of switches determining which metadata enhancement routines are run and to enable pointing to source/target root folder


USAGE:

First import tags from your files into a nominated database:

```cd /root_folder_you_want_to_import_tags_from```

```python /path.to/puddletag/tagfromdb3.py import /path/to/database_you_want_created_incuding_filename_eg_x.db .```


let that run - it'll take a while to ingest tags from your FLAC library, writing each file's metatada to a table called alib

run dropbannedtags.py against the same database

```python ~/dropbannedtags.py /tmp/flacs/x.db```


It'll report its workings and stats as it goes.

When it's done the results (changes only) are written to export.db, which can be exported back to the underlying files like so:


```python /path.to/puddletag/tagfromdb3.py export /tmp/flacs/export.db .```

This will overwrite the tags in the associated files, replacing it with the revised tags stored in export.db

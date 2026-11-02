Regenerating 17_printing_reference.pdf
======================================

The PDF is generated from the database, not hand-written. If 17_printing.sql
changes, regenerate rather than edit: every column table, every foreign key,
every constraint and every index in it is read from the PostgreSQL catalog
after the schema has been applied, so what it prints is what the database
actually has.

Unlike 16_docgen, nothing here executes transactions — the document is a
reference, not a transcript — so it can be re-run as often as you like
against the same scratch database.

Requires: PostgreSQL 18 client + a server you can create a scratch DB on,
          libreoffice (headless), ghostscript, python3.

  1.  Stand up a scratch database with the four master tables 17_printing.sql
      references, then apply 10_app_settings.sql (for the settings seed in
      §4.4) and 17_printing.sql:

        initdb -D <datadir> -U vk --auth=trust
        pg_ctl -D <datadir> -o "-k /tmp/nexpg -p 55432 -c listen_addresses=''" start
        psql -h /tmp/nexpg -p 55432 -U vk -d postgres -c "CREATE ROLE postgres SUPERUSER LOGIN"
        createdb -h /tmp/nexpg -p 55432 -U vk printdoc
        psql ... -d printdoc <<'SQL'
          CREATE SCHEMA fixed;
          CREATE TABLE public.companys      (comp_id uuid PRIMARY KEY, comp_name text);
          CREATE TABLE public.branch_master (br_id uuid PRIMARY KEY, br_comp_id uuid, br_name text);
          CREATE TABLE public.user_master   (usr_id uuid PRIMARY KEY, usr_name text);
          CREATE TABLE fixed.device_master  (dev_id uuid PRIMARY KEY, dev_name text);
        SQL
        psql ... -d printdoc -f ../10_app_settings.sql
        psql ... -d printdoc -f ../17_printing.sql

      The socket path MUST be short (/tmp/nexpg). A path under a long
      scratchpad directory exceeds the 107-byte Unix socket limit and the
      server refuses to start.

  2.  python3 printdoc.py           -> writes printdoc.html

      It prints "<n> columns undocumented" and lists them on stderr. That
      number must be zero: every column must resolve either to an entry in
      prose.PROSE or to a suffix in prose.COMMON, so a column added to
      17_printing.sql cannot slip into the document unexplained.

  3.  ./topdf.sh                    -> writes ../17_printing_reference.pdf

      It splits the HTML at the page-break markers, converts each part with
      soffice and merges with gs. Splitting is not optional: LibreOffice
      degrades super-linearly and appears to hang on the whole file.

Files
-----
  printdoc.py   the document: structure, narrative, and the catalog queries
  prose.py      the per-column usage text (COMMON = suffix defaults,
                PROSE = per-column overrides)
  topdf.sh      split, convert, merge
  README.txt    this file

What the document covers
------------------------
  Part 1   the subsystem: the two questions, the seven tables, one render
           end to end
  Part 2   the relationships: the design chain, the reference matrix, every
           foreign key with its referential action and why, the three
           unusual keys, the keys deliberately absent, and index coverage
           verified from the catalog
  Part 3   table by table: every column, its type, default, key and what it
           is for; then every constraint and every index
  Part 4   the vocabularies, the uniqueness rules, the stored-SQL guards,
           and the settings the subsystem reads

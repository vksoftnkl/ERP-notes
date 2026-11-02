Regenerating 16_stock_reference.pdf
===================================

The PDF is generated from the database, not hand-written. If 16_stock.sql
changes, regenerate rather than edit: the field tables in Part 3 are read
from the PostgreSQL catalog, and the results in Part 4 are the real output
of the statements shown.

Requires: PostgreSQL 18 client + a server you can create a scratch DB on,
          libreoffice (headless), ghostscript, python3.

  1.  Edit docreset.sh so SOCK/PORT point at your scratch server.
      It creates database "nexdoc", stubs the master tables that
      16_stock.sql references, applies the schema, and seeds the four
      demonstration items.

  2.  ./docreset.sh

  3.  Refresh the catalog dump that Part 3 is built from:
        psql ... -At -c "SELECT json_agg(x)::text FROM ( ... )" > cols.json
      (the exact query is in the session notes; any dump with keys
       tbl/n/col/typ/notnull/def/gen works)

  4.  python3 gendoc.py          -> writes stockdoc.html
      NOTE: gendoc.py EXECUTES the Part 4 transactions as it runs, so it
      must be run exactly once against a freshly reset database.

  5.  LibreOffice degrades super-linearly on a document this size and will
      appear to hang on the whole file. Split at the page-break markers and
      merge:
        - split stockdoc.html into 5 files at
          '<p style="page-break-before:always; margin:0; font-size:1pt">'
        - soffice --headless --convert-to pdf each part
        - gs -sDEVICE=pdfwrite -sOutputFile=16_stock_reference.pdf part*.pdf

Files
-----
  gendoc.py     the document: structure, narrative, and the worked transactions
  prose.py      the per-column usage text (COMMON = suffix defaults,
                PROSE = per-column overrides). Every column must resolve to
                one of the two; gendoc reports any that do not.
  docseed.sql   the four demonstration items and their tracking policies
  post.sql      t_post(), the stand-in for the service layer
  docreset.sh   drops and rebuilds the scratch database

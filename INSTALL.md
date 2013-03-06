Installing miniSASS Django-CMS project
======================================

This is currently a basic Django-CMS site. No extra Django applications have been added yet. To install the pre-requisites needed for this site to run, do the following:


Create a PostGreSQL database, based on a template that includes the PostGIS functionality.

    createdb -T postgis-template minisass-cms

postgis-template is the name of your postgis-enabled database template.

or if you have postgresql >= 9.1 and PostGIS >= 2.0 then 

    createdb minisass-cms
    psql -c 'CREATE EXTENSION postgis;' minisass-cms



Create the project directory
----------------------------

    mkdir miniSASS
    cd miniSASS/


Create the python virtual environment
-------------------------------------

    virtualenv python
    source python/bin/activate

Install initial system-level dependencies

    apt-get install python-dev libjpeg8-dev libpng12-dev libfreetype6-dev zlib1g-dev
 (or libjpeg62-dev if you are on Ubuntu 12.04 or lower)

If you still get this in the next step when compiling PIL:

    --------------------------------------------------------------------
    *** TKINTER support not available (Tcl/Tk 8.5 libraries needed)
    *** JPEG support not available
    *** ZLIB (PNG/ZIP) support not available
    *** FREETYPE2 support not available
    *** LITTLECMS support not available
    --------------------------------------------------------------------
    To add a missing option, make sure you have the required
    library, and set the corresponding ROOT variable in the
    setup.py script.

then try this or similar: http://www.jayzawrotny.com/blog/django-pil-and-libjpeg-on-ubuntu-1110 (untested)

Install the Django-CMS dependencies
-----------------------------------

    pip install pil==1.1.7
    pip install psycopg2
    pip install Django==1.4
    pip install django-cms
    pip install south
    pip install django-filer
    pip install cmsplugin-filer
    pip install django-reversion==1.6

Checkout this repository from GitHub
------------------------------------

    git clone https://github.com/sokolic/miniSASS.git minisass
    cd minisass/

Now, copy settings.py.templ to settings.py, and set the database credentials correctly.

    cp minisass/settings.py.templ minisass/settings.py
    vim settings.py
    
Change settings to fit your database credentials, etc.

Initialize your database and start the site
-------------------------------------------

    python manage.py syncdb --all
    python manage.py migrate --fake
    python manage.py runserver

You should have a running, but empty, Django-CMS website.


Setting up the spatial data layers
==================================

Rivers and dams
---------------

Download from http://www.dwa.gov.za/iwqs/gis_data/river/rivs500k.html

> shp2pgsql -s 4326 -c -D -I -W LATIN1 wriall500 rivers | psql -d minisass-cms

> shp2pgsql -s 4326 -c -D -I -W LATIN1 dams500g dams | psql -d minisass-cms

Catchments and WMAs
-------------------

download from here: http://www.dwaf.gov.za/Dir_BI/SLIMDownload/%28S%28gd31jnee31s4nwzcqbf1qu20%29%29/Default.aspx

> shp2pgsql -s 4326 -c -D -I -W LATIN1 sde_other_SDE_dprim_conv hca1 | psql -d minisass-cms

> shp2pgsql -s 4326 -c -D -I -W LATIN1 sde_other_SDE_dsec_conv hca2 | psql -d minisass-cms

> shp2pgsql -s 4326 -c -D -I -W LATIN1 sde_other_SDE_dter_conv hca3 | psql -d minisass-cms

> shp2pgsql -s 4326 -c -D -I -W LATIN1 sde_other_SDE_dquat_conv hca4 | psql -d minisass-cms

> shp2pgsql -s 4326 -c -D -I -W LATIN1 DistrictMunicipalities2011.shp | psql -p 5433 -d minisass-cms

> shp2pgsql -s 4326 -c -D -I -W LATIN1 Province_New_SANeighbours.shp | psql -p 5433 -d minisass-cms

> shp2pgsql -s 4326 -c -D -I -W LATIN1 LocalMunicipalities2011.shp | psql -p 5433 -d minisass-cms

Schools
-------

miniSASS sample points
----------------------
We'll load these and pubish as WMS for early development, but might end up publishing directly via Django and OL. This is also not the final schema, just 'as is' from GroundTruth. First save the sample .xlsx file as a dbf after formatting the Data column as 'date with time'. I also had to roundtrip the date as text into a new column and fiddle with locales to get the date as a string in the right order. It arrived as MS decimal date.

> shp2pgsql -n -c -D 'GT0380-Example miniSASS data - DUCT May Day River Walk' sample_temp | psql -d minisass-cms

We then use SQL to generate and populate parts of the final schema from the sample data.

    CREATE TABLE sites 
    (
      gid serial NOT NULL,
      the_geom geometry(Point,4326),
      name character varying(100),
      description character varying(255),
      river_cat character varying(5), --'sandy' or 'rocky', Django list
      CONSTRAINT sites_pk PRIMARY KEY (gid )
    )
    WITH (
      OIDS=FALSE
    );

    CREATE INDEX sites_geom_idx
      ON sites
      USING gist
      (the_geom );

    --SELECT setval('public.sites_gid_seq', 0, true);
    INSERT INTO sites (the_geom,name)
    SELECT ST_PointFromText('POINT('||longitude||' '||latitude||')', 4326), site_name
    FROM sample_temp;

    CREATE TABLE observations 
    (
      gid serial NOT NULL,
      user_id integer,
      flatworms boolean,
      worms boolean,
      leeches boolean,
      crabs_shrimps boolean,
      stoneflies boolean,
      minnow_mayflies boolean,
      other_mayflies boolean,
      damselflies boolean,
      dragonflies boolean,
      bugs_beetles boolean,
      caddisflies boolean,
      true_flies boolean,
      snails boolean,
      score numeric(4,2),
      site integer,
      time_stamp timestamp without time zone,
      comment character varying(255),
      CONSTRAINT observations_pk PRIMARY KEY (gid )
    )
    WITH (
      OIDS=FALSE
    );
    
    INSERT INTO observations (site)
    SELECT gid
    FROM sites;
    
    ALTER TABLE sample_temp ADD UNIQUE (site_name);
    
    UPDATE observations o SET score = s.mini_sass_, time_stamp = s.date_iso,comment = s.comment
    FROM (SELECT s.*,t.comment,t.mini_sass_,t.date_iso::timestamp without time zone FROM sites s INNER JOIN sample_temp t on s.name = t.site_name) s
    WHERE site = s.gid; 
    
    ALTER TABLE observations ADD FOREIGN KEY (site) REFERENCES sites (gid) ON UPDATE NO ACTION ON DELETE NO ACTION;
    
    DROP TABLE sample_temp;

To add a layer to QGIS you can use something like this in DB Manager:

> SELECT s.*,o.comment,o.score,o.time_stamp,o.user_id FROM sites s INNER JOIN observations o on o.site = s.gid
    



---
layout: post
title:  "Tutorial: FreeSWITCH User Directory integration with PostgreSQL"
date:   2021-07-11 06:09:01 +0500
categories: freeswitch postgresql integration
---


## Introduction

FreeSWITCH uses XML for it's configuration, which was a popular choice back in 2008 when FreeSWITCH was founded by Anthony Minessale

However it also supports database and third party system integration for it's core db and configuration files, dialplan etc using modules.

When you deploy FreeSWTICH as an office PBX, you need a structured way of running day to day operations of the PBX, which in case of XML files becomes very cumbersome and sometimes very complex to maintain 

> Before you continute, make sure you have followed the FreeSWITCH installation instruction in our previous post.

In order to serve user directory from a PostgreSQL database we will use following method

1.  mod\_lua bindings for serving User directory, a LUA script will take care of database integration and other details
2.  Users will be stored as a JSON object in a JSON type column, PostgreSQL supports &quot;JSON&quot; columns
3.  3rd party postgres-json-schema extension to validate JSON schema elements

Basic syntax validation of JSON document is done by PostgreSQL server itself, however to avoid pitfalls and ensure data integry we need to validate FreeSWITCH related elements in JSON doc before inserting/updating, unfortunately PostgreSQL doesn&#39;t support schema validation i.e. validation a JSON document against a JSON schema. In order to accomplish this we will use a 3rd party PostgreSQL extension &quot;postgres-json-schema&quot;, once installed it provides us a function `validate_json_schema(schema, data)` that returns true if data validates against the schema and false if not

## Prerequisites

Make sure following requirements are already met

*   FreeSWITCH 1.8 or above is already installed, (mod\_lua is loaded)
*   Core (`switch.conf.xml`) and internal sip profile (`sip\_profiles/internal.xml`) db is already set to ODBC DSN
*   PostgreSQL server v. 12.0  is installed and running

## Installation Steps

### Install PostgreSQL development package

```bash
sudo apt install -y postgresql-server-dev-12
```

### Clone postgres-json-schema repository

```bash
cd /usr/local/src/
git clone https://github.com/gavinwahl/postgres-json-schema.git
```

### Install &quot;`postgres-json-schema`&quot; extension

Change to `postgres-json-schema` directory and run `make install`, the output should look like this

```bash
[root@master ~]# cd postgres-json-schema/
[root@master postgres-json-schema]# make install
/usr/bin/mkdir -p '/usr/pgsql-10/share/extension'
/usr/bin/mkdir -p '/usr/pgsql-10/share/extension'
/usr/bin/install -c -m 644 .//postgres-json-schema.control '/usr/pgsql-10/share/extension/'
/usr/bin/install -c -m 644 .//postgres-json-schema--0.1.0.sql  '/usr/pgsql-10/share/extension/'
```

### Load &quot;postgres-json-schema&quot; extension

Connect to the database to create the “`postgres-json-schema`” extension. Extension creates “`validate_json_schema`” Pl/PgSQL function, which can be called against JSON data column in CHECK constraint. You can also use a GUI tool like PgAdmin4&#39;s Query Tool

```bash
[root@master]# su postgres -
[postgres@master]# psql 
psql (12)
Type "help" for help.

postgres=# \dx
                 List of installed extensions
  Name   | Version |   Schema   |         Description
---------+---------+------------+------------------------------
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)

postgres=# CREATE EXTENSION "postgres-json-schema";
CREATE EXTENSION
```

### Create extensions table for User Directory

Create a table in PostgreSQL to hold FreeSWITCH Directory users, Our LUA script will query this table to fetch user(s)

```sql
CREATE TABLE extensions
(
    extension character varying(10) COLLATE pg_catalog."default" NOT NULL,
    json_data jsonb,
    CONSTRAINT extensions_pkey PRIMARY KEY (extension),
)
```

Create a CHECK constraint on above created table, this CHECK constraint will validate json\_data field on INSERT/UPDATE using provided schema, You may add more properties to schema anytime

```sql

ALTER TABLE extensions ADD CONSTRAINT data_is_valid CHECK (validate_json_schema('{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "title": "FreeSWITCH JSON Schema for User Directory",
    "description": "In case you are going to put your SIP User inside a postgres jsonb field, you will need this schema", 
    "type": "object",
    "properties": {
      "id": { "type": "integer" },
      "params": {
        "type": "object",
        "properties": {
          "password": { "type": "string" },
          "a1-hash": { "type": "string" },
          "dial-string": { "type": "string" },
          "vm-password": { "type": "string" },
          "vm-enabled": { "type": "string" },
          "vm-mailto": { "type": "string" },
          "vm-email-all-messages": { "type": "string" },
          "vm-notify-all-messages": { "type": "string" },
          "vm-attach-file": { "type": "string" },
          "jsonrpc-allowed-methods": { "type": "string" },
          "jsonrpc-allowed-event-channels": { "type": "string" }
        },
        "oneOf": [text
          { "required": [ "password" ] },
          { "required": [ "a1-hash" ] }
        ],
        "additionalProperties": false
      },
      "variables": {
        "type": "object",
        "properties": {
          "user_context": { "type": "string" },
          "callgroup": { "type": "string" },
          "sched_hangup": { "type": "string" },
          "toll_allow": { "type": "string" },
          "accountcode": { "type": "string" },
          "nibble_account": { "type": "string" },
          "origination_caller_id_name": { "type": "string" },
          "origination_caller_id_number": { "type": "string" },
          "effective_caller_id_name": { "type": "string" },
          "effective_caller_id_number": { "type": "string" },
          "outbound_caller_id_name": { "type": "string" },
          "outbound_caller_id_number": { "type": "string" }
        },
        "required": [ "user_context", "callgroup", "accountcode" ],
        "additionalProperties": true
      }
    },
    "required": [ "id", "params", "variables" ]
}', json_data));
```

Now, postgres will reject any INSERTs or UPDATES to the table that set data to anything not valid against its schema, Insert a sample user record to test if everything is working

```sql
INSERT INTO public.extensions(
	extension, json_data)
	VALUES (7001, '{
    "id": 7001,
    "params": {
        "password": "7001",
        "vm-mailto": "g.mustafa@example.com",
        "dial-string": "the dial dialstring",
        "vm-password": "123"
    },
    "variables": {
        "callgroup": "sales",
        "accountcode": "33579",
        "user_context": "default"
    }
}');
```

### Configure `mod_lua` bindings

The file `autoload_configs/lua.conf.xml` is used in the default FreeSWITCH setup. Here is a minimum configuration file, it will fetch User directory from Lua script.

```xml

<configuration name="lua.conf" description="LUA Configuration">
  <settings>
    <param name="xml-handler-script" value="user_directory.lua"/>
    <param name="xml-handler-bindings" value="directory"/>
  </settings>
</configuration>
```

> When you edit lua.conf.xml, giving the reloadxml command from the FreeSWITCH console is not sufficient to recognize the &quot;xml-handler-script&quot; parameters, you have to restart FreeSWITCH.

### Create LUA script

Our LUA Script is the heart of this integration task. This script will be executed whenever FreeSWITCH requires a user information e.g for registering and calling. Create/Open a new file `user_directory.lua` inside `/usr/local/freeswitch/scripts` in your favorite editor and insert following content

```lua
-- Load required libraries 
package.path = '/usr/local/freeswitch/scripts/json.lua'
json = require("json")

package.path = '/usr/local/freeswitch/scripts/common.lua'
common = require("common")

-- temp logging
-- freeswitch.consoleLog("debug", "[xml_handler] KEY_NAME: " .. XML_REQUEST['key_name'] .. "");
-- freeswitch.consoleLog("debug", "[xml_handler] KEY_VALUE: " .. XML_REQUEST['key_value'] .. "");
-- freeswitch.consoleLog("debug", "[xml_handler] SECTION: " .. XML_REQUEST['section'] .. "");
-- freeswitch.consoleLog("debug", "[xml_handler] TAG_NAME: " .. XML_REQUEST['tag_name'] .. "");
-- freeswitch.consoleLog("debug", "[xml_handler] Params: " .. params:serialize() .. "");

-- Get User from Request
user = params:getHeader("user")
domain_name = params:getHeader("domain")
event_calling_func = params:getHeader("Event-Calling-Function")

-- Get database connection handl3e
db = freeswitch.Dbh("odbc://freeswitch:freeswitch:freeswitch!")
-- db = freeswitch.Dbh("pgsql://hostaddr=127.0.0.1 dbname=freeswitch user=freeswitch password='freeswitch!' options='-c client_min_messages=NOTICE' application_name='freeswitch'")
-- check if we are connected to the database
assert(db:connected())

if common.is_user_directory_function(event_calling_func) then
	query = string.format("select extension, json_data::text as json_data from extensions where extension = '%s'", user)
	freeswitch.consoleLog("debug", "User query: " .. query);

	function fetch_user_object(row)
		json_data = row.json_data
		extension = row.extension
	end

	db:query(query, fetch_user_object)

	if (json_data == nil) then
		XML_STRING = common.XML_STRING_NOT_FOUND
	end

	if (json_data) then 
	    freeswitch.consoleLog("debug", "JSON:Data -> " .. json_data)
	    lua_value = json.parse(json_data) -- Get lua object from json object

	    lua_value['params']['dial-string']  = "{^^:sip_invite_domain=${dialed_domain}:presence_id=${dialed_user}@${dialed_domain}}${sofia_contact(*/${dialed_user}@${dialed_domain})},${verto_contact(${dialed_user}@${dialed_domain})}";
	    call_group = lua_value['variables']['call_group']
	    if call_group == nil then
		    call_group = user
	    end
            local xml = {}
            table.insert(xml, [[<document type="freeswitch/xml">]]);
            table.insert(xml, [[  <section name="directory">]]);
            table.insert(xml, [[    <domain name="]] .. domain_name .. [[">]]);
            table.insert(xml, [[      <groups>]]);
            table.insert(xml, [[        <group name="]] .. call_group .. [[">]]);
            table.insert(xml, [[           <users>]]);
            table.insert(xml, [[             <user id="]] .. user .. [[">]]);
            for block, bvalue in pairs(lua_value) do 
                if type(bvalue) == 'table' then
                        table.insert(xml, [[               <]].. block  ..[[>]]);
                for tag, tvalue in pairs(bvalue) do
                            table.insert(xml, [[                 <]].. string.sub(block, 1, -2)  ..[[ name="]] .. tag .. [[" value="]] .. tvalue .. [["/>]]);
                end
                        table.insert(xml, [[               </]].. block  ..[[>]]);
                end
            end
            table.insert(xml, [[             </user>]]);
            table.insert(xml, [[           </users>]]);
            table.insert(xml, [[        </group>]]);
            table.insert(xml, [[      </groups>]]);
            table.insert(xml, [[    </domain>]]);
            table.insert(xml, [[  </section>]]);
            table.insert(xml, [[</document>]]);

            XML_STRING = table.concat(xml, "\n");
	end
else
		XML_STRING = common.XML_STRING_NOT_FOUND
end -- user directory event calling funcs

freeswitch.consoleLog("debug", XML_STRING)
```

## Testing

Register a SIP endpoint/phone with following credentials to check if our `PostgreSQL` and `FreeSWITCH User Directory` integration via a LUA Script is successful. Open `fs_cli` for checking verbose logs and errors User: `7001` Password: `7001`
# pgsql-lited
A lightweight native PostgreSQL driver written in D

The goal is a native driver that re-uses the same buffers and the stack as much as possible,
avoiding unnecessary allocations and work for the garbage collector

Interface compatible with mysql-lited


## notes
- NOT ready for production use yet
- supports all PostgreSQL types with conversion from/to D native types
- results can be retrieved through a flexible and efficient callback interface
- socket type is a template parameter - currently only a vibesocket is implemented
- extended protocol not yet supported


## example
```d
import std.stdio;

import pgsql;


void usedb() {

	// use the default pgsql client - uses only prepared statements
	auto client = new PgSQLClient("host=sql.moo.com;user=root;pwd=god;db=mew");
	auto conn = client.lockConnection();

	// simple select statement
	User[] users;
	conn.execute("select name, email from users where id > ?", 13, (PgSQLRow row) {
		users ~= row.toStruct!User;
	});


	// batch inserter - inserts in packets of 128k bytes
	auto insert = inserter(conn, "users_copy", "name", "email");
	foreach(user; users)
		insert.row(user.name, user.email);
	insert.flush;

	//struct inserter - insterts struct directly using the field names and UDAs.
	struct Info {
		string employee = "employee";
		int duration_in_months = 12;
	}

	struct InsuranceInfo {
		int number = 50;
		Date started = Date(2015,12,25);
		@ignore string description = "insurance description";
		Info info;
	}

	struct BankInfo {
		string iban;
		string name;
		@as("country") string bankCountry;
	}

	struct Client {
		@as("name") string clientName = "default name";
		@as("email") string emailAddress = "default email";
		@as("token") string uniuqeToken = "default token";
		@as("birth_date") Date birthDate = Date(1991, 9, 9);
		@ignore string moreInfoString;
		InsuranceInfo insurance;
		BankInfo bank;
	}

	auto dbClient = conn.fetchOne!Client("select * from client limit 1");

	assert(client.serialize == dbClient.serialize)

	//batch insert struct array
	Client[] clients = [Client(), Client(), Client()];
	insert.rows(clients);

	// passing variable or large number of arguments
	string[] names;
	string[] emails;
	int[] ids = [1, 1, 3, 5, 8, 13];
	conn.execute("select name from users where id in " ~ ids.placeholders, ids, (PgSQLRow row) {
		writeln(row.name.peek!(char[])); // peek() avoids allocation - cannot use result outside delegate
		names ~= row.name.get!string; // get() duplicates - safe to use result outside delegate
		emails ~= row.email.get!string;
	});


	// another query example
	conn.execute("select id, name, email from users where id > ?", 13, (size_t index /*optional*/, PgSQLHeader header /*optional*/, PgSQLRow row) {
		writeln(header[0].name, ": ", row.id.get!int);
		return (index < 5); // optionally return false to discard remaining results
	});


	// structured row
	conn.execute("select name, email from users where length(name) > ?", 5, (PgSQLRow row) {
		auto user = row.toStruct!User; // default is strict.yesIgnoreNull - a missing field in the row will throw
		// auto user = row.toStruct!(User, Strict.yes); // missing or null will throw
		// auto user = row.toStruct!(User, Strict.no); // missing or null will just be ignored
		writeln(user);
	});


	// structured row with nested structs
	struct GeoRef {
		double lat;
		double lng;
	}

	struct Place {
		string name;
		GeoRef location;
	}

	conn.execute("select name, lat as `location.lat`, lng as `location.lng` from places", (PgSQLRow row) {
		auto place = row.toStruct!Place;
		writeln(place.location);
	});


	// structured row annotations
	struct PlaceFull {
		uint id;
		string name;
		@optional string thumbnail;	// ok to be null or missing
		@optional GeoRef location;	// nested fields ok to be null or missing
		@optional @as("contact_person") string contact; // optional, and sourced from field contact_person instead

		@ignore File tumbnail;	// completely ignored
	}

	conn.execute("select id, name, thumbnail, lat as `location.lat`, lng as `location.lng`, contact_person from places", (PgSQLRow row) {
		auto place = row.toStruct!PlaceFull;
		writeln(place.location);
	});


	// automated struct member uncamelcase
	@uncamel struct PlaceOwner {
		uint placeID;			// matches placeID and place_id
		uint locationId;		// matches locationId and location_id
		string ownerFirstName;	// matches ownerFirstName and owner_first_name
		string ownerLastName;	// matches ownerLastName and owner_last_name
		string feedURL;			// matches feedURL and feed_url
	}
}
```

## todo
- add proper unit tests
- implement more statements
- implement extended protocol
- make vibe-d dependency optional

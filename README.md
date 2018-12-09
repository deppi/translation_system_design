Translations System Design
# tables:

Tables to be implemented in a relational database, such as SQL


## user:

u_id       - UUID (PK)
first_name - varchar(50)
last_name  - varchar(50)
password   - varchar(100) (hashed value stored)
locale_id  - UUID (FK)
ut_id      - UUID (FK)


## user_type:

ut_id	  - UUID (PK)
user_type - varchar(20) // can be engineer, translator, or user


## locale:
// contains the support locales by the system

locale_id - UUID (PK)
locale    - varchar(4) (like EN_US, FR_CA, ...)


## phrase: 
// stores all phrases that exist in system
// for phrases that can have pluralizations the phrase_plural field will not be NULL
// substitutions inside the phrase can be made by substituting '%s' inside the phrase
// for example 'Welcome to Wealthsimple, %s' is a phrase that can replace %s with user's name.

id            - UUID (PK)
ph_id         - UUID
phrase        - varchar(2000)
phrase_plural - varchar(2000) // used for when this phrase has a plural variant
locale_id     - UUID (FK)


## view: 

v_id       - UUID (PK)
view_name  - varchar(100)


## view_phrase: 
// each view in the system will have an associated v_id.
// for each view you can gather the collection of phrases needed to render that view

id        - UUID (PK)
v_id      - UUID (FK)
ph_id     - UUID (FK)



# services:



## phrase service:

routes:

### GET:
/phrase_group/{ph_id} - returns phrases specified by {ph_id}
	
	if > 1 phrases matched:
		STATUS: 200
		[{
			id1,
			ph_id1,
			phrase,
			phrase_plural, // (if present)
			locale_id
		}, ...]
	else:
		STATUS: 404 phrases not foudn

/phrase/{locale_id},{id} - returns phrase specified by {id}

	if phrase present:
		STATUS: 200
		{
			id,
			ph_id,
			phrase,
			phrase_plural, // (if present)
			locale_id
		}
	else:
		STATUS: 404 phrase not found

/phrases/{locale_id},{ph_id1},{pd_id2},{...}, - returns list of phrases specified by locale_id and list of ph_id

	if 1 or more phrases matched:
		STATUS: 200
		{
			matched: x -- where X is the number of matched phrases
			phrases: [{
				id1,
				ph_id1,
				phrase,
				phrase_plural, // (if present)
				locale_id
			}, ...]
		}
	else:
		STATUS: 404 phrases not found

### POST:
/phrase - creates new phrase

	body: 
	{
		id,
		ph_id,
		phrase,
		phrase_plural, // optional
		locale_id
	}

	response:
	if valid input:
		STATUS: 200 
		{
			ph_id
		}
	else:
		STATUS: 405 invalid input

### PUT:
/phrase - updates phrase with specified fields

	body: 
	{
		id,
		ph_id,
		phrase, // optional
		phrase_plural, // optional
		locale_id // optional
	}

	response:
	if valid input:
		STATUS: 200 
		{
			ph_id
		}
	else:
		STATUS: 404 if ID not matched
		STATUS: 405 if invalid input

### DELETE:
/phrase - deletes phrase by specified id

	body:
	{
		id
	}

	response:
	if valid input:
		STATUS: 200 
		{
			id
		}
	else:
		STATUS: 404 ID not matched



## view service:

routes:

### GET:
/view/{v_id} - returns view matched by v_id

	if v_id present:
		STATUS: 200 
		{
			v_id
			view_name
		}
	else:
		STATUS: 404 ID not found 

/views/{page_num},{skip} - returns 50 views by page, skip is the page number you are on. (for view dashboard described below)

	response:
	[{
		v_id1,
		view_name
	},{
		v_id2,
		view_name
	} ...]

### POST:
/view - creates a new view entry
	
	body:
		{
			v_id
			view_name
		}

	response:
	if valid input:
		STATUS: 200 
		{
			v_id
		}
	else:
		STATUS: 405 invalid input

### PUT:
/view - updates view entry specified by v_id

	body:
		{
			v_id
			view_name
		}

	response:
	if valid input:
		STATUS: 200 
		{
			v_id
		}
	else:
		STATUS: 404 if ID not matched
		STATUS: 405 if invalid input

### DELETE:
/view

	body:
	{
		v_id
	}

	response:
	if valid input:
		STATUS: 200 
		{
			v_id
		}
	else:
		STATUS: 404 ID not matched




## view_phrase service:

routes:

### GET: 
/view_phrase/{id} - gets view phrase by ID

	if v_id present:
		STATUS: 200 
		{
			id
			v_id
			ph_id
		}
	else:
		STATUS: 404 ID not found 

	/view_phrase_aggregation/{v_id},{locale_id} - returns list of phrases based on v_id and locale

	1) reaches out to view service GET /view to validate v_id exists
		- if doesn't exist return 404 - Invalid v_id
	2) grabs all phrase IDs associated with v_id from view_phrase table
	3) reaches out to phrase service GET /phrases to get all related phrases to the view and locale.
		- if no matches, return 404 - No phrases matched by locale_id and v_id
	
	success returns:
	200
	[{
		id1,
		ph_id1
		phrase
		phrase_plural
		locale_id
	}, ...]

### POST: 
/view_phrase

	body:
	{
		id
		v_id
		ph_id
	}

	response:
	if valid input:
		STATUS: 200 
		{
			id
		}
	else:
		STATUS: 405 invalid input

### PUT: 
/view_phrase

	body:
		{
			id
			v_id
			ph_id
		}

	response:
	if valid input:
		STATUS: 200 
		{
			id
		}
	else:
		STATUS: 404 if ID not matched
		STATUS: 405 if invalid input

### DELETE: 
/view_phrase

	body:
		{
			id
		}

	response:
	if valid input:
		STATUS: 200 
		{
			id
		}
	else:
		STATUS: 404 ID not matched



# front ends:

- Each front end page that gets served on an app or webpage can be specified by a view.
- Every time a view gets rendered, the /view_phrase_aggregation route can be called to get all required phrases for a given locale. The responses to this can be cached by this service if necessary.
	- to fully render a front end, first you could use a template renderer such as jinja to replace ph_id's in HTML with the actual phrases. THEN for each ph_id that has a replacement, use a 
		{ pd_id -> [ replacement1, replacement2, ... ] }
	mapping to replace each %s with required data.
- Developers can easily add new phrases by either directly manipulating the table, or preferrably making requests to /phrase POST route.
- Analysts can have a dashboard that displays 50 views per page. you can click on a view and then load all phrases for that view (using /view, and /phrase_group routes) to display everything that is existing for that view. analysts can update (/phrase PUT) phrases and add translations.
# John's notes

These are my notes on the codebase.  I'm writing them as I a go along to try to
help me understand the codebase.  I may eventually try to turn them into
real documentation.

## Background

This is basically the entire system for the startup company Health Concierge.

The company provides a service that helps people find the best care for their particular situation.

The system is split into 3 phases:

1. Pre-call
2. Call
3. Post-call

## Pre-call

The pre-call phase begins when the client contacts the company with a request. Clients can initiate cases through a web interface where they provide key information about their healthcare needs.

A case record is created in the system containing:
- Case number (unique identifier)
- Client name/nickname
- Description of needs
- Location
- Any specific requirements or preferences

[Case model definition: `back/datap/models/doctor_match.py:Case`]
[Case creation endpoint: `back/datap/routing/front_doctor_match_router.py:post_cases_new`]
[Frontend case creation: `front/src/lib/components/doctor-match/CaseEditor.svelte`]

The system then performs an automated search process to identify potential healthcare providers. This search process includes:

1. Provider Discovery:
   - Uses NPI (National Provider Identifier) database for initial provider discovery
   - Supports multiple search strategies (v1, v2, v3) with different discovery and enrichment methods:
     - v1: Google + Bing enrichment
     - v2: NPI discovery + Bing enrichment
     - v3: NPI discovery + IPRoyal enrichment

[Search implementation: `back/datap/tasks/doctor_match_tasks.py:doctor_match_precall_search_v1` and `doctor_match_precall_search_v2`]
[Search query model: `back/datap/schema/doctor_match.py:PrecallSearchQuery`]

2. Provider Enrichment:
   - Gathers additional information about each provider
   - Collects data like:
     - Contact information
     - Office hours
     - Website details
     - Ratings and reviews from multiple platforms (Healthgrades, WebMD, Yelp, RateMDs, ZocDoc)
     - Insurance information
     - Availability

[Provider info gathering: `back/datap/services/doctor_match/postcall/provider_info.py:get_provider_info`]
[Website search: `back/datap/services/doctor_match/postcall/institute_website.py`]
[Ratings and reviews: `back/datap/services/doctor_match/postcall/ratings_reviews_service.py:get_all_ratings_reviews`]

3. Provider Grouping:
   - Organizes providers based on various criteria
   - Prioritizes based on factors like:
     - Distance
     - Ratings
     - Specializations
     - Gender preferences

[Priority configuration: `back/datap/schema/doctor_match.py:PrecallSearchQuery.priorities`]

## Call

Once all of the information from the pre-call stage is gathered, the calling bot is triggered through the system's CallingbotRouter. The bot uses a structured approach to gather information:

1. Each call is assigned a unique call ID
2. The bot is provided with:
   - Provider's phone number
   - Company introduction script
   - List of specific questions to ask
   - Context about the client's insurance and requirements

[Call initiation: `back/datap/services/doctor_match/callingbot/client.py:CallingbotClient.start_call`]
[Call handling: `back/datap/ws/handlers/call.py:CallHandler`]
[Call data model: `back/datap/services/e2e_client.py:initiate_call`]

The bot calls each provider and conducts a structured interview, collecting responses about:
- New patient processes
- Insurance acceptance
- Availability
- Costs
- Specific services or treatments
- Any other relevant criteria for the client's case

[Question processing: `back/datap/services/doctor_match/postcall/llm_modules.py`]
[Constants for questions: `back/datap/util/post_call_constants.py`]

## Post-call

After the bot has gathered information from all providers, the system processes and analyzes the collected data to prepare a comprehensive report for the client. This includes:

1. Data Processing:
   - Verification of provider information
   - Analysis of responses
   - Compilation of ratings and reviews
   - Address and contact information validation

[Data processing: `back/datap/services/doctor_match/postcall/data_processing.py:process_list`]
[Provider info processing: `back/datap/services/doctor_match/postcall/providers.py`]

2. Report Generation:
   - Creates both internal and client-facing versions of reports
   - Organizes providers into "Top Recommendations" and "Alternatives"
   - Includes key information such as:
     - Provider contact details and office hours
     - Insurance compatibility
     - Availability
     - Cost information
     - Ratings and reviews summary
     - Distance from client's location

[Report service: `back/datap/services/doctor_match/postcall/report_service.py`]
[Client report generation: `front/src/lib/components/doctor-match/GenerateClientReportPanel.svelte`]
[Report constants: `back/datap/util/post_call_constants.py:TOP_RECOMMENDATIONS` and `ALTERNATIVES`]

The final report is stored in Google Drive, with separate versions maintained for internal records and client distribution.
[Report copy creation: `back/datap/services/doctor_match/postcall/report_service.py:create_client_copy`]

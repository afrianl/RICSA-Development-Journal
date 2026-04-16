# RICSA Development Journal
## 2026-04-06
- Found the API to render citation page images
- Created a presentation to be presented to Mas Ivo, containing these slides:
	- Development progress
	- Development challenges/blockers
	- What's next
- Listed which API endpoints are going to be used in the app:
	- List Dataset
	- Agent Convo
	- Upload File to Dataset
	- Update Document
	- Delete Document
	- Get Page Image

## 2026-04-07
- Starts research on Google Drive file syncing w/ RAGFlow
- File syncing problem found:
	- Files won't start parsing
	- Files WILL start parsing, just after every 7 hours or so
	- One solution is to check the logs in the machine running RAGFlow

## 2026-04-08
- Found the true causes of Google Drive file syncing problems with RAGFlow:
	- File sync only takes newly modified files on the drive which was/were created between the last sync and the current sync
	- The file sync works wonderfully on local machine, yet only syncs once every 7 hours on the dev server. Might be an env-related issue
- Working on the fix for the aforementioned issues
- Start writing NODIN to SKAI for LHA samples for testing purposes

## 2026-04-09
- Researched LLMs to see which one is best. Current candidate:
	- Gemma 4, refer to [the Gemma 4 document](Gemma_4.md)
	- Qwen 3
- Google Drive file sync has been fixed
- Found out RAGFlow's Dataset Separation does not do anything in creating RBAC
- Found a workaround for the RBAC by using the `/api/v1/retrieval` endpoint instead (actually goated tho idk why i didn't use this one)

## 2026-04-10
- Tested Google Cloud Storage file sync
- GCS file sync succeeds, but has yet to be able to process any bigger PDFs, timeout is the most probable cause 
- Added more timeout duration to handle larger files
- Timeout duration increment gives no avail

## 2026-04-13
- Found the root problem of larger files not processable, caused by nginx data transfer limiting
- Built a small and temporary web app to test out:
	- RBAC
	- Session management
- Concluded that the best workaround to cover both points is to create two distinct chat assistants, one for each clearance level (secret and generic)
- Nevermind, that's stupid because RAGFlow has an issue with chat assistant ownership (whoops)
- Tried out using one singular api key as the entry point and let ragflow manage the user id and session ids

## 2026-04-14
- GCS file upload problem is now solved by reversing the file storage process.
	- Instead of pushing the raw PDF into GCS then processing the files in the GCS into RAGFlow, we push the files into RAGFlow then let RAGFlow forward it into GCS (insane trick thx mas dafa)

## 2026-04-15
- Added file upload API into the temporary web app to test the implementation
- Held a meeting with ASA to double check the API structure

## 2026-04-16
- Updated the example API structure to follow ASA's ERD structure
- Created [the API REFERENCE document](API_REFERENCE.md) as a documentation/reference for ASA to use
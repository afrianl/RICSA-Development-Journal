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
	- Gemma 4, refer to [[./Gemma 4|the Gemma 4 document]]
# Definitions of "done"

A user story (e.g. product feature) or bugfix is considered "done" when the following conditions are met (unless extended or redefined in a particular project).


## "Done" for front-end

1. Functionality validated for overall UX/UI consistency by the project keeper
2. Functionality validated as meeting the need by the requester, or users representative if no particular requester
3. Covered by automated tests (if applicable; should be item #1 in test-first implementations)
4. Client data migration (if applicable) implemented and covered by tests
5. Build succeeds (i.e. all tests green, no code linting errors, etc.) locally and on the build server
6. Implementation validated by a peer
7. Version number up-to-date
8. Documentation (if applicable) up-to-date
9. Change log (if applicable) up-to-date
10. Everything committed and pushed to source control
11. Marketing notified to prepare communication to users/public if relevant


## "Done" for libraries and back-end

1. Automated tests cover the added functionality
2. Functionality validated as meeting the need by the requester, or library/API consumers representative if no particular requester
3. Data migration (if applicable) up-to-date and covered by tests
4. Build succeeds (i.e. all tests green, no code linting errors, etc.) locally and on the build server
5. Implementation validated by a peer
6. Version number is up-to-date
7. Documentation is up-to-date
8. Change log (if applicable) is up-to-date
9. Everything committed and pushed to source control

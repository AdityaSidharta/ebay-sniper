# TODO

- Use uv rust and ty instead of mypy and pylint and flake8
- Do we still need fastapi in this architecture? Do we need API Handler in this architecture, since we are using API Gatewway. Replace FastAPI with Lambda PowerTool
- Do we still need KMS in this architecture?
- What about the other functionalities? Don't we need a lambda for that?
- Don't need real time update on the bid, it will only be refreshed whenever the user open or refresh the page
- Remove the need of Redis in the architecture for the rate limiting purpose
- How do we make sure that a user is not able to access another user's data / perform actions for other user's account?
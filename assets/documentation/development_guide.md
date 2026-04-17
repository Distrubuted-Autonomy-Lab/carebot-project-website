# Carebot Chatbot Development Guide
Author: Walt Wilber

## Making changes to the codebase
The carebot repo is structured as a standard Django web app, with an HTML/CSS frontend and a Python backend. Pertinent files to each aspect of the application can are listed below:

### 1. Frontend 
- `root\app\chat\templates\chat` contains the HTML for all webpages.
- `root\app\chat\static\chat` contains the JS and CSS styling for webpages.

### 2. Backend
- `root\app\chat` contains the Python code for the app. Chat logic is contained in `views.py`.

### 3. Container
- The dockerfile is stored in `root/chat`, and scripts for docker containerization, such as `docker-compose.yml` and `docker-entrypoint.sh` are in `root`.

To see your changes reflected on the app, use `docker compose -f [scripts] down` and `docker compose -f [scripts] up` to restart the app. If changes were made to static files, the whole app will need to be rebuilt for the changes to appear.

## Dev Environment
- The app is written in Python, HTML, CSS, and JavaScript. 
- Compilers are included in the image built by the dockerfile. This includes python 3.12-slim.
- Build management is handled through the Docker composition, and will be handled automatically through the `docker compose` command included in the README.
- Required dependencies can be found in `requirements.txt`

## Backlog
- The project backlog and associated pull requests are handled through Github projects, and can be found in the main repo.
- Future groups may consider updating APIs:
    - Upgrading the Gemini API to a non-deprecated version
    - Using a more robust text-to-speech API

## Style Expectations
- The app generally follows the color scheme of University of Alabama online resources: red, white, and gray.
- The end users are persons with limited tech experience and potential visual or motor impairments, so simple, large, and clear GUI elements are necessary.

## Testing Guide
- Testing scripts have been prepared to test the functionality of the APIs used by the app. For instructions on how to run these scripts, check the testing guide.
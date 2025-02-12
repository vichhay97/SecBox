# SecBox
SecBox tool; a lightweight, container based malware analysis sandbox.
Requires Python version 3.9.

## Env Files
All necessary env files are supplied within the following components. They are set up for local execution and must be altered, should an alternative setup be desired.

## Frontend Setup
The frontend requires [Node 16.X](https://www.stewright.me/2022/01/tutorial-install-nodejs-16-on-ubuntu-20-04/) and
 [Yarn](https://classic.yarnpkg.com/lang/en/docs/install/#debian-stable). To install the dependencies go to 
```
├── SecBox
│   ├── app
```           
and run `npm install`.

Loading the app.env file is necessary before execution.

The frontend can be deployed locally via the command `npm run serve`.

## Backend Setup
The backend requires [python 3.9](https://www.python.org/downloads/release/python-390/) and [pip](https://pip.pypa.io/en/stable/installation/).
All dependencies required for the backend can be installed in
```
├── SecBox
│   ├── api
``` 
with `pip install -r requirements.txt`.

api.env needs to be loaded before running the backend with.

```
python3 webapp_api.py
```

## Host Setup
In order to set up the host on a machine, run:

```
sudo ./setup.sh
sudo ./setup_bazel_gvisor.sh
```

Afterwards, load the necessary environment variables from the file called 'host.env'.

The host can then be run from the host directory by running:

```
sudo -E python3 host.py
```


## Configuration
Configuration happens through the respective .env files for the respective system components.


## Demo Video
A demo is available on YouTube:
https://www.youtube.com/watch?v=lkEE3iQvFtk

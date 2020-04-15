* [Operator](/operator) - Contains scripts, YAML template files, and projects to deploy the SAS Event Stream Processing metering server 
and the ESP operator. Follow the README in this location to deploy a command line version of the SAS Event Stream Processing environment.

The following Docker images are deployed from this location:
  * SAS Event Stream Processing metering server
  * ESP operator
  * Open source postgres database
  * Open source filebrowser to manage the persistent volume


* [Single-User Clients](/single_user_clients) - Contains scripts, YAML template files, and projects to deploy SAS Event Stream Processing 
graphics clients.  Do not deploy the single-user clients until *after* you have deployed the [Operator](/operator). You must run scripts from this directory in the same K8 namespace as that of the operator.

The following Docker images are deployed from this location: 
  * SAS Event Stream Processing Studio
  * SAS Event Stream Processing Streamviewer
  * SAS Event Stream Manager

* [Multi-User Clients](/multi_user_clients) - Contains scripts, YAML template files, and projects to deploy SAS Event Stream Processing 
graphics clients.  Do not deploy the multi-user clients until *after* you have deployed the [Operator](/operator). You must run scripts from this directory in the same K8 namespace as that of the operator.

The following Docker images are deployed from this location: 
  * SAS Event Stream Processing Studio
  * SAS Event Stream Processing Streamviewer
  * SAS Event Stream Manager
  * Open source UAA server
  * SAS Oauth2 proxy server

Each of these subdirectories contain README files with more specific, detailed instructions.
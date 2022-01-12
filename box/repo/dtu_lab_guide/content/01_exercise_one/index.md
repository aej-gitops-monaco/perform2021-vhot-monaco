## First introduction

Dynatrace OneAgent is already installed to the VM and is monitoring 3 applications. In this exercise we will begin by creating an automatic tagging rule inside of our Dynatrace tenant UI. We will then use the Dynatrace API to pull down the configuration of this tag to build our project files. Once our project structure is complete we will remove our automatic tagging rule within the Dynatrace UI and re-apply the rule using Monaco!

## Prerequisites
1. Log into https://university.dynatrace.com
2. On your university dashboard you should see an event `Gitops for Observability With Monitoring As Code`
3. Select the event and locate `Environments`
4. Each participant will have their own Dynatrace Tenant and Virtual machine. Dynatrace University provides an SSH Terminal directly in your browser for connecting to the vm and executing commands.
5. Open the Dynatrace Tenant link and sign in with the provided credentials
6. Open the terminal and execute the command below to ensure Monaco is properly installed.
   
    ```
    $ monaco
    ```

    The expected output for this command will display available command usage.

7. Locate your VM's public IP address from the university page and open a new browser tab. Navigate to **dashboard.YOUR\_VM\_IP\_HERE.nip.io** this dashboard will provide links to a local Gitea server and Jenkins.

    ![ACE Dashboard](../../assets/images/ace_dashboard.png)


8. All lab instructions will be provided in this document but Monaco documentation can be found on [Github](https://github.com/dynatrace-oss/dynatrace-monitoring-as-code)




### Step One - Create Tagging Rule in Dynatrace UI

First we will create a tag that identifies the owners of specific process groups.

1. On the left hand navigation panel expand `Manage` and select `Settings`
2. Navigate to `Tags` -> `Automatically applied tags`
3. Select `Create Tag`
4. Set `Tag name`:
  
    ```
    Owner
    ```

5. Select `Add a new rule`
6. Set `Optional tag value`:
   
    ```
    {ProcessGroup:Environment:owner}
    ```

7. Rule applies to `Process Groups`
8.  Our condition will be `owner (environment) - Exists`
9.  Check the box `Apply to all services provided by the process groups`
10. Click `Create rule`
11. Click `Save changes`
![Owner Tag Config](../../assets/images/TagRule-UI.png)

You can now filter Process Groups by tag `Owner`
![Owner PG](../../assets/images/Owner-PG.png)

### Step Two - Review Monaco Project structure in Gitea

We will review the monaco Project structure and will use Gitea to edit any necessary files in later exercises.
1. Open your Monaco HOT dashboard and open Gitea if you haven't already.
2. Open the `perform` repo and navigate to `monaco` -> `01_exercise_one`, open the `projects` folder.
3. Open and edit the `environments.yaml` file
    1. Environments are defined in the environments.yaml consisting of the environment url and the name of the environment variable to use for the API token. Multiple environments can be specified. Remove all content in `environments.yaml` and copy the block below and paste into the empty file:
    
        ```
        perform:
          - name: "perform"
          - env-url: "YOUR_TENANT_URL_GOES_HERE"
          - env-token-name: "DT_API_TOKEN"
        ```

    2. For security reasons you should not place your environment token directly in a file.
    3. Update the env-url value to your Dynatrace Tenant address (ensure there is no trailing `/` at the end of the URL)
    4. Commit the changes. The file should now look like this:

      ![Owner PG](../../assets/images/environmentsyaml_completed.png)
    

4. Return to the `projects` folder. Here you'll find a folder called `perform` (referenced in our environments.yaml) that contains another folder called `auto-tag` with two files `auto-tag.json` and `auto-tag.yaml`. 
    1. Both files contain only placeholders for the repository. We'll need to update them.
    2. The .json file is a template used for our API payload we plan to send for tagging.
    3. The .yaml file is used for the configuration we want to populate our .json with. The .yaml file can contain multiple configurations that can build different tag names and rules as monaco will iterate through each config and apply it to the template. In this scenario we're simply applying a single automatic tagging rule called 'Owner'.

### Step Three - Build the Monaco JSON from the Dynatrace API
A great way to start building your monaco project is based off existing Dynatrace configuration. Even if you're starting with a fresh Dynatrace environment it may be worth creating a sample configuration in the UI first. This way we'll use the Dynatrace API to extract the properties of the configuration we'd like to use in monaco. From there you can use your configuration yaml file to add additional configuration.

***If you already have a Dynatrace API token please skip ahead to item #4***

1. To use the Dynatrace Swagger UI we need to aquire our API token which is stored in a Kubernetes Secret. 
2. From the Dynatrace University terminal execute the below command to retrieve your token.

    ```bash
    kubectl -n dynatrace get secret oneagent -o jsonpath='{.data.apiToken}' | base64 -d | xargs echo
    ```

    NOTE: The browser terminal may not display the token in it's own line. Be sure to copy only the part prior to your username

3. Paste your token into a notepad for later reference.
4. Open your Dynatrace tenant. Select your profile icon and choose `Configuration API`
5. Once in the Dynatrace Config API UI, select `Authorize`
    ![Tokenauth](../../assets/images/Tokenauth.png)
6. Paste your token into the token field and click `Authorize`. Close the modal on success.
7. Find the `Automatically Applied Tags` endpoint and select it to expand.

    ![Auto-tagendpoints](../../assets/images/autotagendpoints.png)

Next we need to get the Tag ID of `Owner` we manually created earlier.

1. Expand `GET /autoTags`
2. Click `Try it out`
3. Click `Execute`
4. Scoll down to the response body. Copy the id of the `Owner` tag.

    ![Tagid](../../assets/images/TagID.png)

Next we'll use the GET for /autoTags/{id} endpoint

1. Expand `GET /autoTags/{id}`
2. Click `Try it out`
3. Paste the id into the required `id` field
4. Set the boolean flag `includeProcessGroupReferences` to `true`
5. Click `Execute`
6. Scroll down to the response body

    ![Tagconfigjson](../../assets/images/Tagconfigjson.png)

7. Copy the entire response body to your clipboard.
8. Open Gitea and navigate to `monaco` -> `01_exercise_one` -> `projects` -> `perform` -> `auto-tag` and open the `auto-tag.json` file.
9. Edit the file

    ![editjson](../../assets/images/Editjson.png)

10. Remove the placeholder and paste the copied response body from the Dynatrace API output.
11. Once the JSON is pasted into the file, remove lines 2-8. Lines 2-8 are identifiers of the existing configuration that are not accepted when creating a new configuration in the next step. The desired file contents can also be copied from these instructions below.
    
    ```json
    {
      "name": "Owner",
      "rules": [
        {
          "type": "PROCESS_GROUP",
          "enabled": true,
          "valueFormat": "{ProcessGroup:Environment:owner}",
          "propagationTypes": [
            "PROCESS_GROUP_TO_SERVICE"
          ],
          "conditions": [
            {
              "key": {
                "attribute": "PROCESS_GROUP_CUSTOM_METADATA",
                "dynamicKey": {
                  "source": "ENVIRONMENT",
                  "key": "owner"
                },
                "type": "PROCESS_CUSTOM_METADATA_KEY"
              },
              "comparisonInfo": {
                "type": "STRING",
                "operator": "EXISTS",
                "value": null,
                "negate": false,
                "caseSensitive": null
              }
            }
          ]
        }
      ]
    }
    ```

12. Commit the file

### Step Four - Build the Config YAML

1. In the same repo navigate to `01_exercise_one` -> `projects` -> `perform` -> `auto-tag`
2. Open and edit `auto-tag.yaml`
3. Remove the placeholder
4. Copy the contents below and paste into the YAML file.

    ```yaml
    config:
        - tag-owner: "auto-tag.json"
      
    tag-owner:
        - name: "Owner"
    ```

    NOTE: Monaco will require the configuration YAML to always contain a `name` attribute.

5. Commit your changes

The config yaml tells Monaco which configuration JSON to apply. You can supply addtional configuration names with seperate JSON files. 

Each config name has a set of properties to apply to the JSON template. In our case we're telling Monaco to use the auto-tag.json file for our tag-owner configuration. Then the tag-owner configuration has a value called name.

Next we will update our auto-tag.json file to be more dynamic and use environment variables to populate the tag name. This way we could structure our config yaml to iterate through multiple tag names and configurations. 


### Step Five - Modify Auto-Tag.json

1. Open and edit the `auto-tag.json` file under `monaco` -> `01_exercise_one` -> `projects` -> `perform` -> `auto-tag`
2. On line 2 replace `Owner` with the snippet below. 

    ```json
    {{ .name }}
    ```
    Our JSON template is now using the property `name` dynamically that is defined in the config yaml. This practice provides flexibility to define multiple configs or values from other sources and populate them dynamically.

    Expected json file contents:
    ```json
    {
      "name": "{{ .name }}",
      "rules": [
        {
          "type": "PROCESS_GROUP",
          "enabled": true,
          "valueFormat": "{ProcessGroup:Environment:owner}",
          "propagationTypes": [
            "PROCESS_GROUP_TO_SERVICE"
          ],
          "conditions": [
            {
              "key": {
                "attribute": "PROCESS_GROUP_CUSTOM_METADATA",
                "dynamicKey": {
                  "source": "ENVIRONMENT",
                  "key": "owner"
                },
                "type": "PROCESS_CUSTOM_METADATA_KEY"
              },
              "comparisonInfo": {
                "type": "STRING",
                "operator": "EXISTS",
                "value": null,
                "negate": false,
                "caseSensitive": null
              }
            }
          ]
        }
      ]
    }
    ```

3. Commit your changes

### Step Six - Clone repo to VM and execute Monaco

Now that our project files are defined for a tagging rule we'll need to manually delete our existing tag rule in the Dynatrace UI. Then we'll clone our Gitea repo to our VM and exectute monaco from our project structure to re-apply our tag.

1. Open the Dynatrace UI and navigate to `Settings`
2. Open `Tags` -> `Automatically applied tags` and delete the tag called `Owner`
3. Save changes
4. Open the Dynatrace University Terminal
5. cd into the peform directory
   
    ```bash
    $ cd ~/perform
    ```
  
6. Execute the following command to pull down our changes to the remote repository.
   
    ```bash
    $ git pull
    ```

7. cd into the Monaco projects folder
   
    ```bash
    $ cd ./monaco/01_exercise_one/projects
    ```

    ***We're now ready to see Monaco in action!***

8. For the purposes of this training environment we'll use an environment variable to supply Monaco with our Dynatrace token. For security reasons this is not recommended in live environments. Consider storing the token safely such as a secret or credential vault.

9. Create a local environment variable called `DT_API_TOKEN` and input your token value we created earlier.
In case you need to get your API token again execute: (ensure not to copy the linux user)

    ```bash
    $ export DT_API_TOKEN=$(kubectl -n dynatrace get secret oneagent -o jsonpath='{.data.apiToken}' | base64 -d)
    ```

1.  Execute Monaco dry run (-d is the dry run flag) which will validate our configuration before applying the configuration to the Dynatrace tenant. The -e flag tells Monaco which environment we'd like to execute this config for. The project does not need to be specified as Monaco will automatically search the current directory for the project folder. Monaco does allow a -p flag to explicitly specify a project directory.

    ```bash
    $ monaco -d -e ./environments.yaml
    ```

    Monaco should execute and you should not see any errors

    ![monacodryrun](../../assets/images/monacodryrun.png)

2.  Remove the -d flag to execute all configurations in the project!

    ```bash
    $ monaco -e environments.yaml
    ```
3.   Check your Dynatrace tenant for the `Owner` tag to be recreated.

      ![Owner Tag](../../assets/images/Ownertagui.png)

## ***Congratulations on completing exercise one!***





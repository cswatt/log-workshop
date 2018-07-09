# Workshop Exercise 2: Implementing logging best practices

The second exercise is made upon a dummy application that generates different logs with different formats. It logs them in different sources (file, UDP)

If not done already, [enable the log-management product in your Datadog application][6].

## Launch the script

In order to perform this exercise we must spawn a vagrant VM, it allows us to work in a controlled environment, abstracting all potential OS related issue and making this work flow repeatable.

The OS distribution and version used for this exercise is `bento/ubuntu-16.04`. 

1. Start your vagrant VM:
  
  `vagrant up`

2. Connect to your Workshop vagrant box:
  
  `vagrant ssh`

3. Export your [Datadog API Key](https://app.datadoghq.com/account/settings#api):

   `export DD_API_KEY=<DD_API_KEY>`

   We export the Datadog API key in our current shell in order to be able to call it at any time with `$DD_API_KEY`. 

4. Go in the `/vagrant/workshop/exercise_2/` folder to start the exercise:

  `cd ~/vagrant/workshop/exercise_2/`

5. Launch the dummy script: 

    `python main.py &`

## Installing the Agent

To start gathering logs from our system we can use any log-shipper, but in order to benefit from the advantages displayed in the exercise 1 of this workshop we strongly advise you to use the Datadog Agent. To get the Datadog agent:

1. Connect to your [Datadog Application][2]
2. Install the Datadog Agent on your machine:

    ```
    DD_API_KEY=$DD_API_KEY bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh)"
    ```

Once installed you should see this into your shell:

```
Your Agent is running and functioning properly. It will continue to run in the
background and submit metrics to Datadog.

If you ever want to stop the Agent, run:

    sudo systemctl stop datadog-agent

And to run it again run:

    sudo systemctl start datadog-agent
```

This means that the Datadog agent is up and running and is ready to be configured.

## Gathering Data

We have 3 types of log: **full text** | **JSON** | **UDP**, we need to configure our agent accordingly ([Log collection documentation][5])

To do:

* Enable log collection in `/etc/datadog-agent/datadog.yaml` by setting `logs_enabled: true`

* Create a file `workshop.d/conf.yaml` in the ` /etc/datadog-agent/conf.d/` folder with the following content:

```
logs:

  - type: file
    path: /vagrant/workshop/exercise_2/text_log.log
    service: text_log
    source: dummy_app
    sourcecategory: custom
    tags: workshop:exercise_2, type:text_log

  - type: file
    path: /vagrant/workshop/exercise_2/json_log.log
    service: json_log
    source: dummy_app
    sourcecategory: custom
    tags: workshop:exercise_2, type:json_log

  - type: file
    path: /var/log/datadog/*.log
    service: datadog-agent
    source: syslog
    sourcecategory: agent
    tags: workshop:exercise_2, type:datadog-agent

  - type: udp
    port: 4242
    service: udp_log
    source: dummy_app
    sourcecategory: custom
    tags: workshop:exercise_2, type:udp_log
```

* Give access to the folder to the DD agent `sudo chown -R dd-agent:dd-agent /var/log/datadog`.
 
* Restart your agent `sudo service datadog-agent restart`.

* Check if everything is running smoothly: `sudo datadog-agent status`.

## Exploring the data

Go into your [log-explorer view][6] and check that your logs are here.

## Processing data

### Full text logs

1. [Create a  pipeline][7] to parse full text log **ONLY** (Set-up the correct filter on the pipeline `service:text_log`)

    ![text_pipeline](/workshop/exercise_2/images/text_pipeline.png)

2. Implement a Grok parser in this pipeline: 

    ```
    rule %{date("yyyy-MM-dd HH:mm:ss.SSSSSS"):date} %{word:severity} %{word:user} connected to %{notSpace:http.url} it took %{number:http.response_time:scale(0.001)} s and ended up with the %{number:http.status_code} status code user agent used was %{data:http.user_agent}
    ```

    ![Text log parser](/workshop/exercise_2/images/text_log_grok_parser.png)

3. Implement a severity remapping on the main log status with [the log status remapper][9]

    ![text_log_remapping_severity](/workshop/exercise_2/images/text_log_remapping_severity.png)

The final pipeline should look like this:

![text_log_final_pipeline](/workshop/exercise_2/images/text_log_final_pipeline.png)

and transform this log:

```
2018-07-02 09:04:22.533142 EMERGENCY Jane connected to http://my.website_1.com/path/number/3/?query=param_1&var=foo_2 it took 4329000 s and ended up with the 503 status code user agent used was Mozilla/5.0%2520(X11;%2520Linux%2520x86_64;%2520rv:60.0)%2520Gecko/20100101%2520Firefox/60.0
```

into this log:

```json
{
    "date": 1530522262533,
    "http": {
        "response_time": 4329,
        "status_code": 503,
        "url": "http://my.website_1.com/path/number/3/?query=param_1&var=foo_2user_agentMozilla/5.0%2520(X11;%2520Linux%2520x86_64;%2520rv:60.0)%2520Gecko/20100101%2520Firefox/60.0"
    },
    "severity": "EMERGENCY",
    "user": "Jane",
}
```

### JSON log

1. [Create a  pipeline][7] to parse JSON log **ONLY** (Set-up the correct filter on the pipeline `service:json_log`)

2. Use [attribute remappers][10] to remap  `user_agent_bis` on `httpuser_agent` and `url` on `http.url`.

The final pipeline should look like this:

![json_log_final_pipeline](/workshop/exercise_2/images/json_log_final_pipeline.png)

and transform this log:

```

```

into this log:

```

```

### UDP log

1. Clone the Text log pipeline and renaming it into the UDP log.

2. Change the pipeline filter value to `service:udp_log` to apply this Pipeline only to UDP logs

### Main processing pipeline 
The current pipelines should look like this:

![3_pipelines](/workshop/exervice_2/images/3_pipelines.png)

Now that all the different source of logs have a unified format, let's create a main processing pipeline to enhance all our logs:

1. Create the pipeline:
    ![main pipeline configuration](/workshop/imagesmain_pipeline_conf.png)

2. Parse the `http.url` attribute with the [URL processor][11]
    ![url parser](/workshop/exervice_2/images/url_parser.png)

3. Parse the `http.user_agent` attribute with [User Agent Processor][12]
    ![user_agent parser](/workshop/exervice_2/images/user_agent_parser.png)

4. Create an attribute categories on the status code [with the categories processor][13]

    ![category processor](/workshop/exervice_2/images/category_processor.png)

The final pipeline should look like this:

![main_processing_pipeline](/workshop/exervice_2/images/main_processing_pipeline.png)

## Facets

Now that we have all our logs parsed and enhanced we can start adding our attributes as [facet][14].

**Note**: that only new logs attributes are taken into account by the facets.

Add the `user`, `duration`, and `http.url` attributes as facet.

1. Click on the attribute you want to define as a facet:
    ![creating facet](/workshop/exervice_2/images/creating_facet.png)

2. Configure your Facet:
    ![configuring facet](/workshop/exervice_2/images/configuring_facet.png)

Facet can be used to filter, either on the string value or on a double/int range.

## Search 

Here are different search to try out:

1. Search for all logs with `status_code` above 400

    `@http.status_code:>400`

2. Find all log from the user `John`

    `@user_name:"john"`
 
## Log Graph

Looking at raw logs like this can be useful, but if you want to make your log talk switching to log-graph will allow you to do some TI (Technical Inteligence) with the [Log graphs][15] It's like BI, but instead it's on Log :)

Try to display:

*  The top `http.url`
*  The top `http.url` according to the duration
*  The top `http.url` from `user:john` with a `4xx` or `5xx` status code

## Monitor

Let's monitor those data, enter a query and use it to define a monirot.
We could monitor for instance the amount of `5xx` or `4xx` that are generated by our stack at any given moment.


[2]: https://app.datadoghq.com/
[5]: https://docs.datadoghq.com/logs/log_collection/
[6]: https://app.datadoghq.com/logs
[7]: https://docs.datadoghq.com/logs/processing/
[8]: https://docs.datadoghq.com/logs/#reserved-attributes
[9]: https://docs.datadoghq.com/logs/processing/#log-status-remapper
[10]: https://docs.datadoghq.com/logs/processing/#remapper
[11]: https://docs.datadoghq.com/logs/processing/#url-parser
[12]: https://docs.datadoghq.com/logs/processing/#useragent-parser
[13]: https://docs.datadoghq.com/logs/processing/#category-processor
[14]: https://docs.datadoghq.com/logs/explore/#facets
[15]: https://docs.datadoghq.com/logs/graph/
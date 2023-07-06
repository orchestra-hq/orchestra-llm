# orchestra-llm
Repo for generating Orchestra ymls using LLMs



# Pipelines

A pipeline is essentially one enormous json file. A pipeline represents a statement or a declaration of what the user
wants to happen. Within orchestra, every possible configuration of the json files is mapped to specific function calls.
This allows the user to specify the execution of incredibly complex operations with the smallest amount of information
necessary. 

A pipeline object can be executed - this leads to the creation of a pipeline run, which is a log of how the specific
pipeline execution goes.

A pipeline must also be created in order for any pipeline tasks to be specified. A pipeline has certain compulsory
and non-compulsory arguments that are necessary for creation. These can be summarised by the below which is python ORM code:

```
class Pipeline(Base):
    __tablename__ = "pipelines"
    pipelineguid = Column(UUID(as_uuid=True), primary_key=True, index=True)
    clientguid = Column(UUID(as_uuid=True), index=True)
    pipelinename = Column(String, index=False)
    error_message = Column(String, index=False)
    created_time = Column(TIMESTAMP, index=False)
    scheduletypecode = Column(Integer, index=False)
    num_tasks = Column(Integer, index=False)
    tags = Column(JSON, index=False)
    frequency = Column(Integer, index=False)
    frequencytypecode = Column(Integer, index=False)
    statuscode = Column(
        Integer, ForeignKey("internal_status_codes.status_code"), index=False
    )
    starttime = Column(TIME, index=False)
    pipeline_stages = relationship("PipelineStage", back_populates="pipeline")
    client = relationship("Client", back_populates="pipelines")
    status_name = relationship("InternalStatusCodes", back_populates="pipelines")
```

pipelineguid: not required: a unique identifier for the pipeline
clientguid: required: the client the pipeline relates to. This would typically relate to a team.
pipelinename: required: should be unique across pipelines for a given client
error_message: not required: a field that displays the error message if there is one for the latest pipeline run for the pipeline
created_time: not required: created on creation; the time it was created.

# Pipeline Tasks

A pipeline task represents an atomic unit of work that can be executed. Tasks are categorised by integration and by integration job. An integration represents a third party API, or a similar logical concept (A specific type of microservice would be an example). An integration job represents a specific thing to execute within the integration - these will typically be triggered by HTTP / API Calls but are, in reality, extremely complex. For example, for a data ingestion tool like Fivetran (the integration is Fivetran), there is one integration job; a fivetran_sync. A fivetran_sync updates data from a source (e.g. Salesforce) to a sink (E.g. Snowflake) and does so intelligently and per the configuration in fivetran. An obvious pre-requisite for this job is that it needs to already exist (and you need to know its ID, which represents a parameter for the fivetran job). 

This approach is very powerful, because on Orchestra's side, we only require the following pieces of information to start executing the job:

- Integration Name
- Integration Job Name
- Job ID

The job, is, in reality, very complex. Fivetran handles a huge amount of complexity under the hood. Orchestra also does more than just trigger the sync via API. Upon pipeline element creation, Orchestra configures a webhook in Fivetran and sets up an endpoint in Orchestra to listen for the webhook. Upon receipt of the webhook, Orchestra also fetches supplementary data about the Fivetran sync that is then enriched and cross-referenced to data assets. It also creates records of the fivetran runs (in the context of a pipeline run) which can be surfaced. Because this is done using the Orchestra framework, these additional settings and steps of logic you would otherwise need to build yourself if using an Orchestration-only open-source package (like Airflow). This declarative form saves engineers lots of time.

A pipeline task has strict requirements for creation, depending on the type it is. For most tasks, you will not get away with creating them without some kind of ID parameter.

```
class TaskCreate(TaskBase):
    integration: constants.integration_literals
    integration_job: constants.integration_job_literals
    async_: Union[Literal_noBug["yes", "no"], None] = Field(default="no")

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        if (
            self.integration == "fivetran"
            and self.integration_job == "fivetran_sync_all"
        ):
            if "connectionid" not in self.parameters:
                raise HTTPException(
                    status_code=400, detail={"message": "No connection id provided"}
                )
        if self.integration == "hightouch" and self.integration_job == "hightouch_sync":
            if "syncid" not in self.parameters:
                raise HTTPException(
                    status_code=400, detail={"message": "No Hightouch sync id provided"}
                )
        if self.integration == "census" and self.integration_job == "census_sync":
            if "syncid" not in self.parameters:
                raise HTTPException(
                    status_code=400, detail={"message": "No Census sync id provided"}
                )
        if self.integration == "http" or self.integration == "sdk":
            if not self.secondary_configuration_id:
                if "base_url" not in self.parameters:
                    raise HTTPException(
                        status_code=400,
                        detail={"message": "Provide base_url or secondary_config_id"},
                    )
                if "method" not in self.parameters:
                    raise HTTPException(
                        status_code=400,
                        detail={"message": "Provide method or secondary_config_id"},
                    )
                if "endpoint" not in self.parameters:
                    raise HTTPException(
                        status_code=400,
                        detail={"message": "Provide endpoint or secondary_config_id"},
                    )
```

# Pipeline Stages

A pipeline stage is a user-facing feature that, in the backend, is just a special type of task.

Suppose you have to ingest 100 different tables before carrying out a data transformation task. This makes sense; you don't want to start transforming a table that selects from different sources until you know all those sources are updated. 

Rather than specify a dependency for every single task (i.e. saying the transformation task depends on 100 tasks) you can group the 100 tasks as a pipeline stage. The stage is an object that aggregates data about the 100 tasks and updates accordingly. For example, you might want the pipeline stage to block the pipeline if a list of 10 specific core, non-negotiable tasks fail. Perhaps, depending on how it fails, you want different things to happen (maybe you have core vs. non-core transformations; the non-core transformations get to run even if the pipeline stage fails a core ingestion but none of the others). Dependent tasks depend on the pipeline stage.

In reality, the engine simply creates another task in between the data ingestion tasks and the transformation task. This task is a special task; a pipeline stage task. When it is ready to execute, instead of being sent to an integration router to be executed, it simply executes itself (i.e. updates itself to complete) and takes the required engine (kick off the next tasks, kill the pipeline etc.). This is powerful because it allows users to avoid specifying lots of annoying dependencies and makes dependency graphs more digestable, without over-complicating the underlying architecture of the core engine.

## Task types

Integration: Fivetran
Integration Job: Fivetran sync all
Description: syncs all data for a given connection
Parameters:
    connectionid: the unique identifier for a fivetran sync; is two words separated by an underscore

Example json

```
{
    "integration": "fivetran":
    "integration_job":"fivetran_sync_all":
    "connectionid":"ambush_prettiest"
}
```
Integration: dbt
Integration Job: dbt run job
Description: runs a dbt job that is in dbt cloud
Parameters:
    runid: the unique identifier for a dbt job; normally 6 digits

{
    "integration": "dbt":
    "integration_job":"dbt_runjob":
    "connectionid":"123456"
}

Integration: census
Integration Job: census syncs
Description: runs a sync that is in census cloud
Parameters:
    runid: the unique identifier for a census sync; normally 5 digits

{
    "integration": "census":
    "integration_job":"census_sync":
    "connectionid":"12345"
}

Integration: hightouch
Integration Job: hightouch syncus
Description: runs a sync that is in hightouch cloud
Parameters:
    runid: the unique identifier for a hightouch sync; normally 5 digits
    
{
    "integration": "hightouch":
    "integration_job":"hightouch_sync":
    "connectionid":"12345"
}

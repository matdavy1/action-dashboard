<template>
  <v-card
    class="mx-auto"
    max-width="1000"
  >
    <v-container fluid >
      <v-row dense>
        <v-col
          v-for="workflow in workflows"
          :key="workflow.workflow"
          :cols="workflow.flex"
        >
         <v-card class="card" :color="getColor(workflow.status)" dark>
            <div class="build-name" v-text="workflow.workflow"></div>
            <div class="time" v-text="workflow.createdAt"></div>
            <div v-if="dataReady">
              <div class="error-msg" v-text="workflow.errorMessage" ></div>
            </div>
            <v-btn class="btn" outlined color="transparent" @click='clicked(workflow)' ></v-btn>
          </v-card>
        </v-col>
      </v-row>
    </v-container>
  </v-card>
</template>

<script>
import axios from "axios";
import jsZip from "jszip";
import findIndex from "lodash-es/findIndex";

export default {

   sockets: {
        updatedRun(run) {
            console.log("updatedRun runId: " + run.runId);
            const index = findIndex(this.runs, { workflowId: run.workflowId, branch: run.branch });
            if (index >= 0) {
                this.$set(this.runs, index, run);
            } else {
                this.runs.push(run);
            }
        },
    },

  data: () => ({
    workflows: [],
    token: "",
    dataReady: false

  }),

  mounted() {
    this.setupData();
  },
created() {
      window.addEventListener('beforeunload', function() {
         this.setupData();
      })
    },

   methods: {
      clicked(workflow) {
        console.log(workflow.errorMessage)
        window.open(workflow.cucumberReportUrl, '_blank').focus();
      },

      setupData(){
        axios.get("/api/initialData").then((result) => { 
          this.workflows = result.data.filter(workflow => workflow.repo === "my-repo");
          var repos = []
          this.workflows.forEach(workflow => {
            workflow['flex'] = 6;
            repos.push(workflow.repo)
          })
          if (this.workflows.length % 2 === 1){ 
            this.workflows[this.workflows.length - 1]['flex'] = 12
          }

          var uniqueRepos = [...new Set(repos)]
              this.setArtifactsForWorkflows(uniqueRepos).then(() => {
            this.dataReady = true
            console.log('dataReady true')
          })
        })
      },

      getArtifactURLForWorkflow(workflow, artifacts){
        return artifacts.find(artifact => artifact.name === workflow.workflow).archive_download_url
      },

      getArtifactBlobForWorkflow(url){
        return axios.get(url,
          {headers:{'Authorization':`token ${this.token}`}, 
          responseType: 'blob'}
        )
      },

      async setArtifactsForWorkflow(blob, workflow){
        await jsZip.loadAsync(blob.data).then(async (zip) => {
          for (const filename of Object.keys(zip.files)) {
            await zip.files[filename].async('string').then(fileData => {
              if(filename.includes('error-messages')){
                workflow['errorMessage'] = fileData;
              }
              else if (filename.includes('cucumber-results')){
                workflow['cucumberReportUrl'] = fileData;
              }
              console.log('setting error message')
            })
          }
        })
      },

      getArtifactsForRepo(repo){                       //TODO:Get owner from env variable
        return axios.get(`https://api.github.com/repos/${this.workflows[0].owner}/${repo}/actions/artifacts`,
        {headers:{'Authorization':`token ${this.token}`}, 
        responseType: 'json'}).then(res => {
          return res.data.artifacts
        })
      },

      async setArtifactsForWorkflows(repos){
        for (const repo of repos){ 
           await this.getArtifactsForRepo(repo).then(async (artifacts) =>  {
            for (const workflow of this.workflows) {
              await this.getArtifactBlobForWorkflow(this.getArtifactURLForWorkflow(workflow, artifacts)).then(async (blob) => {
                 await this.setArtifactsForWorkflow(blob, workflow)
              })
            }
          })      
        }
        
      },

    getColor(status){
      switch (status) {
          case "success":
              return "#0e6b0e";

          case "failure":
              return "#FF0000";

          case "in_progress":
          case "queued":
              return "yellow";

          default:
              return "transparent";
      }
    },
  }
}
</script>

<style scoped>
.card{
  height:50px;
}
.build-name{
  font-size: x-large;
  font-family: Arial, Helvetica, sans-serif;
  text-align: center;
  line-height: 50px;
  vertical-align: middle;
}
.error-msg{
  font-size: small;
  font-family: Arial, Helvetica, sans-serif;
  position: absolute;
  right: 0;
  bottom: 0;
}

.time{
  font-size: small;
  font-family: Arial, Helvetica, sans-serif;
  position: absolute;
  bottom: 0;
}
.btn{
  width: 100%;
  height: 100% !important;
  position: absolute;
  top: 0;
}
</style>
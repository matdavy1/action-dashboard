<template>
  <v-card
    class="mx-auto"
    max-width="1000"
  >
    <v-text-field v-model="search" label="Search" class="mx-4"></v-text-field>
    <v-container fluid >
      <v-row dense>
        <v-col
          v-for="workflow in filteredWorkflows"
          :key="workflow.workflow"
          :cols="workflow.flex"
        >
         <v-card class="card" :color="getColor(workflow.status)" dark>
            <div class="build-name" v-text="workflow.workflow"></div>
            <div class="repo" v-text="workflow.repo"></div>
            <div class="time" v-text="workflow.readableDate"></div>
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
    search: '',
    workflows: [],
    token: "",
    dataReady: false
  }),

  mounted() {
    this.setupData();
  },

  created() {
    window.addEventListener('beforeunload', () => {
        this.setupData();
    })
  },

  computed: {
    filteredWorkflows() {
        let filteredWorkflows = this.workflows.filter(workflow => {
          let search = this.search.toLowerCase()
          return workflow.workflow.toLowerCase().includes(search) || 
          workflow.repo.toLowerCase().includes(search)
        })
        filteredWorkflows.forEach(workflow => {
          workflow['flex'] = 6;
        })
        if (filteredWorkflows.length % 2 === 1){ 
          filteredWorkflows[filteredWorkflows.length - 1]['flex'] = 12
        }
        return filteredWorkflows
      }
  },

   methods: {
      clicked(workflow) {
        window.open(workflow.cucumberReportUrl, '_blank').focus();
      },

      setupData(){
        axios.get("/api/initialData").then((result) => { 
          // this.workflows = result.data.filter(workflow => workflow.repo === "action-dashboard");
          this.workflows = result.data;
          console.log(this.workflows)

          var repos = []
          this.workflows.forEach(workflow => {
            workflow['readableDate'] = this.getReadableDate(workflow.createdAt)
            repos.push(workflow.repo)
          })

          var uniqueRepos = [...new Set(repos)]
          this.setArtifactsForWorkflows(uniqueRepos).then(() => {
            this.dataReady = true
          })
        })
      },

      getArtifactURLForWorkflow(workflow, artifacts){
        return artifacts.find(artifact => artifact.name === workflow.workflow).archive_download_url
      },

      //Should make call from backend: routes.js -> github.js
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
            })
          }
        })
      },

      //Should make call from backend: routes.js -> github.js
      getArtifactsForRepo(repo){                       
        return axios.get(`https://api.github.com/repos/${this.workflows[0].owner}/${repo}/actions/artifacts`,
        {headers:{'Authorization':`token ${this.token}`}, 
        responseType: 'json'}).then(res => {
          return res.data.artifacts
        })
      },

      //Will it be easier to have this as one long method with chaining .then()s so we dont have to use async await everywhere?
      async setArtifactsForWorkflows(repos){
        for (const repo of repos){ 
          await this.getArtifactsForRepo(repo).then(async (artifacts) =>  { 
            if (artifacts.length > 0){//Probably actually need to catch for when workflow has no artifact not just when the repo has none
              for (const workflow of this.workflows.filter(workflow => workflow.repo === repo)) {
                await this.getArtifactBlobForWorkflow(
                  this.getArtifactURLForWorkflow(workflow, artifacts)).then(async (blob) => {
                  await this.setArtifactsForWorkflow(blob, workflow)
                })
              }
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

    getReadableDate(createdAt){
        var endDate = new Date(createdAt);
        var today = new Date();
        var days = parseInt((today - endDate) / (1000 * 60 * 60 * 24));
        var hours = parseInt(Math.abs(today - endDate) / (1000 * 60 * 60) % 24);
        var minutes = parseInt(Math.abs(today.getTime() - endDate.getTime()) / (1000 * 60) % 60);
        var seconds = parseInt(Math.abs(endDate.getTime() - today.getTime()) / (1000) % 60); 

        let readableDate = '' 
         if (days > 0){
          readableDate = days + " day" + (days === 1 ? "" : "s")
        }
        else if (hours > 0){
          readableDate = hours +  " hour" + (hours === 1 ? "" : "s")
        }
        else if (minutes > 0){
          readableDate = minutes + " minute" + (minutes === 1 ? "" : "s")
        }
        else {
          readableDate = seconds + " second" + (seconds === 1 ? "" : "s")
        }
        return readableDate + " ago"
    }
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
.repo{
  font-size: small;
  font-family: Arial, Helvetica, sans-serif;
  position: absolute;
  top: 0;
  left: 0;
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
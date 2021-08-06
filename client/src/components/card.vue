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
            <div class="build-name" v-text="workflow.workflow" >
              

            </div>
            <div class="time" v-text="workflow.createdAt">
              </div>
            <v-btn class="btn" outlined color="transparent"  @click='clicked(workflow)' >

            </v-btn>
          </v-card>
        </v-col>
      </v-row>
    </v-container>
  </v-card>
</template>

<script>
import axios from "axios";
import jsZip from "jszip";

export default {

  data: () => ({
    workflows: [],
    token: "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",
    artifactsForRepo: []

  }),

  mounted() {
    axios.get("/api/initialData").then((result) => { 
      // this.workflows = result.data.filter(workflow => workflow.repo === "my-repo");
      this.workflows = result.data;
      console.log(this.workflows)

      var repos = []
      this.workflows.forEach(workflow => {
        workflow['flex'] = 6;
        repos.push(workflow.repo)
      })
      if (this.workflows.length % 2 === 1){ 
        this.workflows[this.workflows.length - 1]['flex'] = 12
      }

      this.getArtifactsForEachRepo(repos)    
    })
  },

  methods: {

    clicked(workflow) {
        //open to github workflow
      // window.open(`https://github.com/${workflow.owner}/${workflow.repo}/actions?query=workflow%3A${workflow.workflow}`, '_blank').focus();
      console.log(workflow)
      this.getArtifactBlob(this.getArtifactURL(workflow)).then(blob => {
        this.readFromBlob(blob)
      })
    },

    getArtifactURL(workflow){
      var artifacts = this.artifactsForRepo.find(repo => repo.repoName === workflow.repo).artifacts;
      return artifacts.find(artifact => artifact.name === workflow.workflow).archive_download_url
    },

    getArtifactBlob(url){
      return axios.get(url,
        {headers:{'Authorization':`token ${this.token}`}, 
        responseType: 'blob'}
      )
    },

    readFromBlob(blob){
      jsZip.loadAsync(blob.data).then( zip => {
        Object.keys(zip.files).forEach(filename => {
          zip.files[filename].async('string')
          .then(fileData => {
            window.open(fileData, '_blank').focus();
            console.log(fileData)      
          })
        })
      })
    },

    getArtifactsForEachRepo(repos){
      var uniqueRepos = [...new Set(repos)]
      uniqueRepos.forEach(repo => {
        axios.get(`https://api.github.com/repos/${this.workflows[0].owner}/${repo}/actions/artifacts`,
        {headers:{'Authorization':`token ${this.token}`}, 
        responseType: 'json'}).then(res => {
          this.artifactsForRepo.push({'repoName': repo, 'artifacts': res.data.artifacts})
        })
      })
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
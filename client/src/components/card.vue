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

    data () {
        return {
            workflows: null,
            artifacts: []
        }
    },
    mounted() {
      axios.get("/api/initialData")
        .then((result) => { 
            this.workflows = result.data.filter(workflow => workflow.repo === "my-repo");
            this.workflows.forEach(workflow => workflow['flex'] = 6)
            if (this.workflows.length % 2 === 1){ 
              this.workflows[this.workflows.length - 1]['flex'] = 12
            }
            console.log(this.workflows)
        }
      )

    },

    methods: {

      clicked(workflow) {
        //open to github workflow
        // window.open(`https://github.com/${workflow.owner}/${workflow.repo}/actions?query=workflow%3A${workflow.workflow}`, '_blank').focus();

        //open to cucumber report
        axios.get(`https://api.github.com/repos/${workflow.owner}/${workflow.repo}/actions/artifacts`,
          {headers:{'Authorization':'token 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'}, 
          responseType: 'json'})
          .then(res => {
            return res.data.artifacts.find(artifact => artifact.name === workflow.workflow).archive_download_url
          }).then(url => {
            axios.get(url,
              {headers:{'Authorization':'token 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'}, 
              responseType: 'arraybuffer'}
            ).then(buffer => {
            jsZip.loadAsync(buffer.data)
            .then( zip => {
              Object.keys(zip.files).forEach(filename => {
                zip.files[filename].async('string')
                .then(fileData => {
                  window.open(fileData, '_blank').focus();
                  console.log(fileData)      
                })
              })
            })
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
<template>
  <v-card
    class="mx-auto"
    max-width="500"
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

export default {

    data () {
        return {
            workflows: null
        }
    },
    mounted() {
        axios.get("/api/initialData")
            .then((result) => { 
                this.workflows = result.data;
                this.workflows[0]['flex'] = 12;
                this.workflows[1]['flex'] = 6;
                this.workflows[2]['flex'] = 6;
                console.log(this.workflows)

            })

    },

    methods: {

        clicked(workflow) {
           window.open(`https://github.com/${workflow.owner}/${workflow.repo}/actions?query=workflow%3A${workflow.workflow}`, '_blank').focus();


            //  this.$emit('click', this.page, event); 
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
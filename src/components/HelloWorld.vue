<template>
  <v-container>
    <v-row class="justify-center pa-12">
      <v-col
          cols="12"
          sm="6"
          md="3"
      >
        <v-text-field
          v-model="projectName"
          label="Project Name"
          outlined
        ></v-text-field>   
      </v-col>
      <v-col
        cols="32"
        sm="16"
        md="8"
      >
        <v-file-input
          v-model="files"
          small-chips
          show-size
          multiple
          counter
          label="Drop Files Here"
        ></v-file-input>
      </v-col>
    </v-row>
    <v-row class="justify-end">
      <v-btn
        color="primary"
        rounded
        @click="uploadFiles"
      >
        Upload Files
      </v-btn>
    
      <v-btn
        color="primary"
        rounded
        @click="readFileNames"
      >
        File Names
      </v-btn>
    </v-row>
  
  </v-container>
</template>

<script>
  import { Storage } from 'aws-amplify'
  export default {
    name: 'HelloWorld',

    data: () => ({
      files: [],
      projectName: null,
      filesNames: [],
    }),
    methods: {
      readFileNames () {
        for (let i = 0; i < this.files.length; i++) {          
          this.filesNames[i] = this.files[i].name
          console.log(this.files[i].type)
        }
        console.log(this.filesNames)
      },
      uploadFiles: async function () {
        for (let i = 0; i < this.files.length; i++) {          
          const upload = await Storage.put(this.filesNames[i], this.files[i], {
            level: "private",
            contentType: this.files[i].type,
               resumable: true,
               completeCallback: (event) => {
                   console.log(`${this.filesNames[i]} Successfully uploaded ${event.key}`);
               },
               progressCallback: (progress) => {
                   console.log(`${this.filesNames[i]} Uploaded: ${progress.loaded}/${progress.total}`);
               },
               errorCallback: (err) => {
                   console.error(`${this.filesNames[i]}: Unexpected error while uploading`, err);
               }
          });
          console.log(upload)
        }
        
      }
    }
  }
</script>

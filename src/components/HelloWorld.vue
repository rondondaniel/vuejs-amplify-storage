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
          @change="onFileInput"
        ></v-file-input>
      </v-col>
    </v-row>
    <v-row class="justify-center pa-14">
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
        @click="testReactivity"
      >
        Test
      </v-btn>
    </v-row>
    <template>
      <v-data-table
        dense
        :headers="headers"
        :items="progressInfo"
        :items-per-page="5"
        hide-default-header
        item-key="name"
        class="elevation-1"
      >
        <template v-slot:item="{ item }">
         <v-row>
          <v-col cols="6" align-self="start">
              {{ item.name }}
          </v-col>
          <v-col cols="3" align-self="center">
            <v-progress-linear
            :background-opacity="0.3"
            :buffer-value="100"
            :height="25"
            :width="150"
            :rounded="true"
            :value="item.percentage"
          >
            <strong>{{ item.percentage }}%</strong>
          </v-progress-linear>
          </v-col>
          <v-col cols="1" align-self="center">
              <v-btn 
                class="mx-2"
                fab
                dark
                small
                color="primary"
                @click="pause(item.id)"
              >
                <v-icon dark>
                  mdi-pause
                </v-icon>
              </v-btn>
          </v-col>
          <v-col cols="1" align-self="center">
              <v-btn 
                class="mx-2"
                fab
                dark
                small
                color="error"
                @click="cancel(item.id)"
              >
                <v-icon dark>
                  mdi-close
                </v-icon>
              </v-btn>
          </v-col>
          </v-row>
        </template>
      
      </v-data-table>
    </template>
  </v-container>
</template>

<script>
  import { Storage } from 'aws-amplify'
  export default {
    name: 'HelloWorld',

    data: () => ({
      files: [],
      projectName: null,
      progressInfo: [],
      fileState: [],
      itemsPerPage: 4,
      headers: [
        {
          text: 'Files',
          align: 'start',
          sortable: false,
          value: 'name',
        },
        { 
          text: 'Percentage (%)', 
          value: 'percentage' 
        },
      ],
    }),
    methods: {
      onFileInput (_files) {
        this.progressInfo = [];
        this.fileState = [];

        for (let i = 0; i < _files.length; i++) {     
          let _value =  { id: i, name: _files[i].name, percentage: 0 }
          this.progressInfo.push(_value)
        }
        console.log(this.progressInfo);
      },
      testReactivity () {
        // want to change 3th file percentage to 50
        let i = 2
        this.progressInfo.splice(i, 1, { id: i, name: this.files[i].name, percentage: 50 })
        console.log(this.progressInfo)
      },
      uploadFiles: async function () {
        for (let i = 0; i < this.files.length; i++) {
          let value =  { id: i, pause: 0, cancel: 0 }
          this.fileState.push(value)

          const upload = await Storage.put(this.projectName + "/" + this.files[i].name, this.files[i], {
            level: "private",
            contentType: this.files[i].type,
            resumable: true,
            completeCallback: (event) => {
              console.log(`${this.files[i].name} Successfully uploaded ${event.key}`);
            },
            progressCallback: (progress) => {
              this.progressInfo.splice(i, 1, { 
                id: i,
                name: this.files[i].name,
                percentage: Math.ceil(progress.loaded / progress.total * 100) 
              }),
              console.log(`${this.files[i].name} Uploaded: ${progress.loaded}/${progress.total}`);
            },
            errorCallback: (err) => {
              console.error(`${this.files[i].name}: Unexpected error while uploading`, err);
            }
          });
          console.log(upload);
          console.log(this.fileState);
          // Check user interaction
          if (this.fileState[i].pause == 1) {
            upload.pause;
            console.log("Pause file:", this.files[i].name);
          } else if (this.fileState[i].cancel == 1) {
            Storage.cancel(upload);
            console.log("Pause file:", this.files[i].name);
          }
        }
      },
      pause (index) {
        console.log("Pause file:", index);
        this.fileState.splice(index, 1, { 
          id: index,
          pause: 1,
          cancel: 0,
        })
      },
      cancel (index) {
        console.log("Cancel file:", index);
         this.fileState.splice(index, 1, { 
          id: index,
          pause: 0,
          cancel: 1,
        })
      },
    }
  }
</script>

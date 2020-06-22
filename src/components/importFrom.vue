<template>
  <div>
    <p class="font-weight-light">
      You must import only the references for your final list of included studies
    </p>
    <b-card no-body>
      <b-tabs id="import-data" card>
        <b-tab title="File upload" active>
          <p class="font-weight-light">
            <b>STEP 1:</b> Export the references for your included studies from your reference management software (e.g. Endnote). You must select RIS as the output style.
          </p>
          <p class="font-weight-light">
            <b>STEP 2:</b> Import the .ris/.txt file into iSoQ.
          </p>
          <b-form-file
            id="input-ris-file-key"
            plain
            @change="loadRefs($event)"></b-form-file>
          <b-button
            block
            :disabled="(fileReferences.length >= 1) ? false : true"
            class="mt-2"
            variant="success"
            @click="saveReferences()">
              Upload
          </b-button>
          <p>
            Reminder: If you later add studies to your review, you can do a second import of these and they will be added to your existing list.
          </p>
        </b-tab>
        <b-tab title="Import from PubMed">
          <b-row>
            <b-col
              sm="6">
              <p class="font-weight-light">
                You can import individual references from PubMed by pasting the references PMID below. The PMID is the 8-digit identification number appearing at the end of the web address for the article on PubMed. Add one PMID per line below and click Find.
              </p>
              <b-form-textarea
                v-model="episte_request"
                placeholder="Ej: 17253524"
                rows="6"
                max-rows="100"></b-form-textarea>
              <b-button
                id="btnEpisteRequest"
                class="mt-2"
                block
                variant="outline-primary"
                @click="EpisteRequest">Find</b-button>
            </b-col>
            <b-col
              sm="6">
              <template
                v-if="episte_loading">
                <div class="text-center text-danger my-2">
                  <b-spinner class="align-middle"></b-spinner>
                  <strong>Loading...</strong>
                </div>
              </template>
              <template
                v-else-if="episte_error">
                <p class="font-weight-light">
                  The reference could not be reached, try again or using other ID
                </p>
              </template>
              <template v-else>
                <ul v-if="episte_response.length">
                  <li v-for="(r, index) in episte_response" :key="index">
                    <b-form-checkbox
                      :id="`checkbox-${index}`"
                      v-model="episte_selected"
                      :name="`checkbox-${index}`"
                      :value="index">
                      {{ r.citation }}
                    </b-form-checkbox>
                  </li>
                </ul>
                <b-button
                  v-if="episte_response.length"
                  variant="outline-success"
                  block
                  @click="saveReferences('EpisteDB')">Import references</b-button>
              </template>
            </b-col>
          </b-row>
        </b-tab>
      </b-tabs>
    </b-card>
    <b-row
      class="mt-3">
      <b-col
        cols="12">
        <b-card
          bg-variant="light">
          <template
            v-if="loadReferences">
            <div class="text-center text-danger my-2">
              <b-spinner class="align-middle"></b-spinner>
              <strong>Loading...</strong>
            </div>
          </template>
          <template v-else>
            <b-row v-if="!local_references.length">
              <b-col
                cols="12">
                  <p>No references have been uploaded</p>
              </b-col>
            </b-row>
            <b-row v-else>
              <b-col
                cols="6"
                class="pt-2">
                <span>
                  You have <b>{{ local_references.length }}</b> references loaded
                </span>
              </b-col>
              <b-col cols="6">
                <b-button
                  block
                  @click="openModalReferencesSingle"
                  variant="outline-primary">
                  View references
                </b-button>
              </b-col>
            </b-row>
          </template>
        </b-card>
      </b-col>
    </b-row>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'importFrom',
  props: {
    references: Array
  },
  data: function () {
    return {
      local_references: this.references || [],
      loadReferences: true,
      pre_references: '',
      fileReferences: [],
      episte_request: '',
      episte_response: [],
      episte_selected: [],
      episte_loading: false,
      episte_error: false,
      msgUploadReferences: ''
    }
  },
  watch: {
    pre_references: function (data) {
      this.fileReferences = []
      const file = data
      const allLines = file.split(/\r\n|\n/)
      // Reading line by line
      const titleTags = ['TI', 'T1', 'T2', 'T3']
      const authorTags = ['AU', 'A1', 'A2', 'A3', 'A4']
      const userDefinable = ['U1', 'U2', 'U3', 'U4', 'U5']
      let base = { authors: [], user_definable: [] }

      allLines.forEach((line) => {
        const _line = line.split('  - ')
        const key = _line[0]
        const content = _line[1]

        if (key === 'TY') {
          base['type'] = content
        }
        if (titleTags.includes(key)) {
          base['title'] = content
        }
        if (authorTags.includes(key)) {
          base['authors'].push(content)
        }
        if (key === 'AB') {
          base['abstract'] = content
        }
        if (key === 'VL') {
          base['volume_number'] = content
        }
        if (key === 'SP') {
          base['start_page'] = content
        }
        if (key === 'EP') {
          base['end_page'] = content
        }
        if (key === 'IN') {
          base['issue_number'] = content
        }
        if (key === 'SN') {
          base['isbn_issn'] = content
        }
        if (key === 'PY') {
          base['publication_year'] = content
        }
        if (key === 'DA') {
          base['date'] = content
        }
        if (key === 'DA') {
          base['date'] = content
        }
        if (key === 'DB') {
          base['database'] = content
        }
        if (key === 'UR') {
          base['url'] = content
        }
        if (key === 'DO') {
          base['doi'] = content
        }
        if (userDefinable.includes(key)) {
          base['user_definable'].push(content)
        }
        if (key === 'ER') {
          this.fileReferences.push(base)
          base = { authors: [], user_definable: [] }
        }
      })
    }
  },
  methods: {
    loadRefs: function (event) {
      const file = event.target.files[0]
      const reader = new FileReader()
      reader.onload = (e) => {
        this.pre_references = e.target.result
      }
      reader.readAsText(file)
    },
    saveReferences: function (from = '') {
      this.loadReferences = true
      let references = ''
      if (!from) {
        references = this.fileReferences
      } else {
        let _r = []
        for (let index of this.episte_selected) {
          let content = JSON.parse(JSON.stringify(this.episte_response[index].content))
          content.publication_year = content.year
          delete (content.year)
          content.isbn_issn = content.publication.ISSN
          if (content.publication.type === 'journal') {
            content.type = 'JOUR'
          }
          _r.push(content)
        }
        references = _r
      }
      let axiosArray = []
      for (let ref of references) {
        ref.organization = this.$route.params.org_id
        ref.project_id = this.$route.params.id
        let newPromise = axios({
          method: 'POST',
          url: `/api/isoqf_references?organization=${this.$route.params.org_id}&project_id=${this.$route.params.id}`,
          data: ref
        })
        axiosArray.push(newPromise)
      }
      axios.all(axiosArray)
        .then((responses) => {
          let cnt = 0
          for (let response of responses) {
            const data = response.data
            this.local_references.push(data)
            cnt++
          }
          const _references = JSON.parse(JSON.stringify(this.local_references))
          this.prefetchDataForExtractedDataUpdate(_references)

          this.msgUploadReferences = `${cnt} references have been added.`
          this.pre_references = ''
          this.fileReferences = []
          this.episte_response = []
          this.$emit('get-references') // this.getReferences(false)
        })
        .catch((error) => {
          console.log('error', error)
        })
    },
    prefetchDataForExtractedDataUpdate: function (references) {
      let _lists = JSON.parse(JSON.stringify(this.lists))
      let _requestFindings = []
      let _requestExtractedData = []

      for (let list of _lists) {
        _requestFindings.push(axios.get(`/api/isoqf_findings/?organization=${this.$route.params.org_id}&list_id=${list.id}`))
      }
      axios.all(_requestFindings)
        .then((responses) => {
          for (let _response of responses) {
            let response = _response.data[0]
            _requestExtractedData.push(axios.get(`/api/isoqf_extracted_data/?organization=${response.organization}&finding_id=${response.id}`))
          }
          this.updateExtractedDataReferences(_requestExtractedData, references)
        })
        .catch((error) => {
          this.printErrors(error)
        })
    },
    updateExtractedDataReferences: function (querys = [], references = []) {
      if (references.length) {
        if (querys.length) {
          axios.all(querys)
            .then((responses) => {
              let item = {}
              let _items = []
              let patchExtractedData = []
              for (let reference of references) {
                item = {
                  'ref_id': reference.id,
                  'authors': this.parseReference(reference, true),
                  'column_0': ''
                }
                _items.push(item)
              }
              for (let _response of responses) {
                let response = _response.data[0]
                for (let _item of _items) {
                  for (let item of response.items) {
                    if (item.ref_id === _item.ref_id) {
                      if (item.column_0.length) {
                        _item.column_0 = item.column_0
                      }
                    }
                  }
                }
                const params = {
                  items: _items
                }
                patchExtractedData.push(axios.patch(`/api/isoqf_extracted_data/${response.id}`, params))
              }
              if (patchExtractedData.length) {
                axios.all(patchExtractedData)
                  .then((responses) => {})
                  .catch((error) => {
                    // this.printErrors(error)
                    console.log(error)
                  })
              }
            })
        }
      }
    },
    EpisteRequest: function () {
      document.getElementById('btnEpisteRequest').disabled = true
      this.episte_loading = true
      this.episte_error = false
      this.episte_response = []
      const allLines = this.episte_request.split(/\r\n|\n/)
      allLines.forEach((line, index) => {
        const instance = axios.create({
          withCredentials: true,
          headers: {
            'Authorization': 'Token token="bcd43ca4789d1d52b28a288828d738c2"'
          }
        })

        instance.get(`https://api.epistemonikos.org/v1/documents/${line}?show=relations`)
          .then((response) => {
            let obj = {}
            obj.citation = response.data.citation
            obj.content = response.data.content
            this.episte_response.push(obj)
            document.getElementById('btnEpisteRequest').disabled = false
            this.episte_loading = false
          }).catch((error) => {
            this.episte_loading = false
            this.episte_error = true
            document.getElementById('btnEpisteRequest').disabled = false
            // this.printErrors(error)
            console.log(error)
          })
      })
    }
  }
}
</script>

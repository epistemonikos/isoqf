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
            v-if="local_loadReferences">
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

    <b-modal
      v-if="selected_list_index >= 0"
      id="modal-references-list"
      ref="modal-references-list"
      title="References"
      @ok="saveReferencesList"
      ok-title="Save"
      ok-variant="outline-success"
      cancel-variant="outline-secondary"
      size="xl"
      scrollable>
      <template v-if="local_references.length">
        <div
          class="mt-2">
          <b-alert
            v-if="showBanner"
            show
            variant="danger">
            <b>Warning!</b> By removing a reference you are modifying the underlining evidence base for this finding and will need to review your GRADE-CERQual assessments. If you remove the reference, the extracted data you inputted from this study to support this finding will be deleted from the GRADE-CERQual Assessment Worksheet.
          </b-alert>
          <b-table
            responsive
            striped
            :fields="[{key: 'checkbox', label: ''}, {key: 'content', label:'Author(s), Year, Title'}]"
            :items="local_modalRefs">
            <template v-slot:cell(checkbox)="data">
              <b-form-checkbox
                :id="`checkbox-${data.index}`"
                v-model="selected_references"
                :name="`checkbox-${data.index}`"
                :value="data.item.id">
              </b-form-checkbox>
            </template>
          </b-table>
        </div>
      </template>
      <template v-else>
        <div
          class="mt-2">
          <p>To select references, first upload your full reference list by clicking "Import References" next to the search bar.</p>
        </div>
      </template>
    </b-modal>

    <b-modal
        id="modal-references"
        ref="modal-references"
        title="References"
        size="xl"
        @ok="getProject"
        @cancel="confirmRemoveAllReferences($event)"
        scrollable
        ok-variant="outline-success"
        ok-title="Close"
        cancel-variant="outline-danger"
        cancel-title="Remove all references"
        :cancel-disabled="disableBtnRemoveAllRefs">
        <template v-if="appearMsgRemoveReferences">
          <b-row>
            <b-col
              cols="12">
              <p>This action will remove all the references</p>
            </b-col>
          </b-row>
          <b-row>
            <b-col
              cols="6">
              <b-button
                block
                @click="removeAllReferences"
                variant="outline-danger">
                Continue
              </b-button>
            </b-col>
            <b-col
              cols="6">
              <b-button
                block
                @click="appearMsgRemoveReferences = false"
                variant="outline-success">
                Cancel
              </b-button>
            </b-col>
          </b-row>
        </template>
        <template v-else>
          <div
            class="mt-2"
            v-if="local_references.length">
            <p>Below are the references you have uploaded.</p>
            <b-table
              sort-by="authors"
              responsive
              hover
              bordered
              borderless
              striped
              :fields="fields_references_table"
              :items="local_references">
              <template v-slot:cell(action)="data">
                <b-button
                  variant="outline-danger"
                  @click="data.toggleDetails">
                  <font-awesome-icon
                    icon="trash"></font-awesome-icon>
                </b-button>
              </template>
              <template v-slot:row-details="data">
                <b-card>
                  <p>You are about to exclude a study from your review. This will delete it, and all associated information, from all tables in iSoQ. If you exclude this study please remember to redo your GRADE-CERQual assessments for all findings that it supported.</p>
                  <p>{{ findRelatedFindings(data.item.id) }}</p>
                  <p>Are you sure you want to delete this reference?</p>
                  <b-container>
                    <b-row>
                      <b-col
                        cols="12"
                        md="6">
                        <b-button
                          block
                          variant="outline-success"
                          @click="data.toggleDetails">No</b-button>
                      </b-col>
                      <b-col
                        cols="12"
                        md="6">
                        <b-button
                          block
                          variant="outline-danger"
                          @click="confirmRemoveReferenceById(data.item.id)">Yes</b-button>
                      </b-col>
                    </b-row>
                  </b-container>
                </b-card>
              </template>
            </b-table>
          </div>
        </template>
      </b-modal>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'importFrom',
  props: {
    references: Array,
    loadReferences: Boolean,
    modalRefs: Array,
    lists: Array,
    charsOfStudies: Object,
    methodologicalTableRefs: Object
  },
  data: function () {
    return {
      local_lists: Array,
      local_references: [],
      local_charsOfStudies: Object,
      local_methodologicalTableRefs: Object,
      local_loadReferences: true,
      pre_references: '',
      fileReferences: [],
      episte_request: '',
      episte_response: [],
      episte_selected: [],
      episte_loading: false,
      episte_error: false,
      msgUploadReferences: '',
      showBanner: false,
      local_modalRefs: [],
      selected_references: [],
      finding: {},
      selected_list_index: null,
      disableBtnRemoveAllRefs: false,
      appearMsgRemoveReferences: false,
      fields_references_table:
        [
          {
            key: 'authors',
            label: 'Author(s)',
            formatter: value => {
              if (value.length < 1) {
                return 'no author(s)'
              } else if (value.length === 1) {
                return value[0].split(',')[0]
              } else if (value.length === 2) {
                return value[0].split(',')[0] + ' & ' + value[1].split(',')[0]
              } else {
                return value[0].split(',')[0] + ' et al.'
              }
            }
          },
          { key: 'publication_year', label: 'Year' },
          {
            key: 'id',
            label: 'Related to finding(s)',
            formatter: value => {
              let findings = []
              for (let list of this.lists) {
                for (let ref of list.raw_ref) {
                  if (ref.id === value) {
                    findings.push(`#${list.isoqf_id}`)
                  }
                }
              }
              return findings.join(', ')
            }
          },
          { key: 'action', label: '' }
        ],

    }
  },
  mounted: function () {
    this.loadData()
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
    },
    loadReferences: function () {
      this.local_loadReferences = this.loadReferences
    },
    references: function () {
      this.local_references = this.references
    }
  },
  methods: {
    loadData: function () {
      this.local_references = this.references
      this.local_modalRefs = this.modalRefs
      this.local_lists = this.lists
      this.local_charsOfStudies = this.charsOfStudies
      this.local_methodologicalTableRefs = this.methodologicalTableRefs
    },
    loadRefs: function (event) {
      const file = event.target.files[0]
      const reader = new FileReader()
      reader.onload = (e) => {
        this.pre_references = e.target.result
      }
      reader.readAsText(file)
    },
    saveReferences: function (from = '') {
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
          let cnt = responses.length
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
          this.emit('print-error', error)// this.printErrors(error)
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
                    // this.emit('print-error', error)// this.printErrors(error)
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
            // this.emit('print-error', error)// this.printErrors(error)
            console.log(error)
          })
      })
    },
    openModalReferencesSingle: function () {
      this.$emit('get-references')// this.getReferences(false)
      this.msgUploadReferences = ''
      this.appearMsgRemoveReferences = false
      this.disableBtnRemoveAllRefs = false
      this.$refs['modal-references'].show()
    },
    saveReferencesList: function () {
      this.loadReferences = true
      this.table_settings.isBusy = true
      const params = {
        references: this.selected_references
      }
      axios.patch(`/api/isoqf_lists/${this.lists[this.selected_list_index].id}`, params)
        .then((response) => {
          this.updateFindingReferences(this.selected_references)
          this.selected_references = []
          this.selected_list_index = null
          this.$emit('get-references')// this.getReferences()
          this.$emit('get-lists')// this.getLists()
        })
        .catch((error) => {
          this.emit('print-error', error)// this.printErrors(error)
        })
    },
    updateFindingReferences: function (references) {
      const params = {
        'evidence_profile.references': references
      }
      axios.patch(`/api/isoqf_findings/${this.finding.id}`, params)
        .then((response) => {
          this.finding = {}
        })
        .catch((error) => {
          this.emit('print-error', error)// this.printErrors(error)
        })
    },
    getProject: function () {
      this.$emit('get-project')
    },
    confirmRemoveAllReferences: function (event) {
      event.preventDefault()
      this.appearMsgRemoveReferences = true
      this.disableBtnRemoveAllRefs = true
    },
    findRelatedFindings: function (refId = null) {
      if (refId) {
        let findings = []
        for (let list of this.lists) {
          for (let ref of list.raw_ref) {
            if (ref.id === refId) {
              findings.push(`#${list.isoqf_id}`)
            }
          }
        }
        if (findings.length) {
          return 'The findings affected are: ' + findings.join(', ')
        } else {
          return 'No findings will be affected.'
        }
      }
    },
    removeAllReferences: function () {
      let _lists = JSON.parse(JSON.stringify(this.local_lists))
      const _charsOfStudies = JSON.parse(JSON.stringify(this.local_charsOfStudies))
      const _assessments = JSON.parse(JSON.stringify(this.local_methodologicalTableRefs))
      const _references = JSON.parse(JSON.stringify(this.local_references))
      let requests = []

      if (Object.prototype.hasOwnProperty.call(_charsOfStudies, 'id')) {
        requests.push(axios.delete(`/api/isoqf_characteristics/${_charsOfStudies.id}`))
      }
      if (Object.prototype.hasOwnProperty.call(_assessments, 'id')) {
        requests.push(axios.delete(`/api/isoqf_assessments/${_assessments.id}`))
      }
      for (let reference of _references) {
        requests.push(axios.delete(`/api/isoqf_references/${reference.id}`))
      }

      let _requestFindings = []
      for (let list of _lists) {
        _requestFindings.push(axios.get(`/api/isoqf_findings?organization=${this.$route.params.org_id}&list_id=${list.id}`))
        list.references = []
        axios.patch(`/api/isoqf_lists/${list.id}`, list)
          .then((response) => {})
          .catch((error) => {
            this.emit('print-error', error)// this.printErrors(error)
          })
      }
      if (_requestFindings.length) {
        axios.all(_requestFindings)
          .then((responses) => {
            let getExtractedData = []
            for (let response of responses) {
              let finding = response.data[0]
              getExtractedData.push(axios.get(`/api/isoqf_extracted_data/?organization=${this.$route.params.org_id}&finding_id=${finding.id}`))
            }
            if (getExtractedData.length) {
              this.getAndDeleteExtractedData(getExtractedData)
            }
          })
          .catch((error) => {
            this.emit('print-error', error)// this.printErrors(error)
          })
      }
      axios.all(requests)
        .then((responses) => {
          this.$emit('get-references')// this.getReferences()
          this.$emit('get-project') // this.getProject()
          this.$emit('reset-data')// this.resetData()
          this.$refs['modal-references'].hide()
        })
    },
    confirmRemoveReferenceById: function (refId) {
      let lists = JSON.parse(JSON.stringify(this.local_lists))
      let _charsOfStudies = JSON.parse(JSON.stringify(this.local_charsOfStudies))
      let _assessments = JSON.parse(JSON.stringify(this.local_methodologicalTableRefs))
      let objs = []

      for (let list of lists) {
        let obj = {id: null, references: []}
        for (let rr of list.raw_ref) {
          if (rr.id !== refId) {
            obj.references.push(rr.id)
          }
          if (rr.id === refId) {
            obj.id = list.id
            objs.push(obj)
          }
        }
      }
      let requests = []

      if (Object.prototype.hasOwnProperty.call(_charsOfStudies, 'id')) {
        if (_charsOfStudies.items.length) {
          let items = []

          for (let item of _charsOfStudies.items) {
            if (item.ref_id !== refId) {
              items.push(item)
            }
          }
          _charsOfStudies.items = items

          requests.push(axios.patch(`/api/isoqf_characteristics/${_charsOfStudies.id}`, _charsOfStudies))
        }
      }

      if (Object.prototype.hasOwnProperty.call(_assessments, 'id')) {
        if (_assessments.items.length) {
          let items = []

          for (let item of _assessments.items) {
            if (item.ref_id !== refId) {
              items.push(item)
            }
          }
          _assessments.items = items

          requests.push(axios.patch(`/api/isoqf_assessments/${_assessments.id}`, _assessments))
        }
      }

      for (let o of objs) {
        requests.push(axios.patch(`/api/isoqf_lists/${o.id}`, {references: o.references}))
      }

      if (requests.length) {
        axios.all(requests)
          .then(axios.spread(function (response) {}))
      }

      axios.delete(`/api/isoqf_references/${refId}`)
        .then((response) => {
          this.$emit('get-references')// this.getReferences(false)
          this.$emit('get-project')// this.getProject()
        })
    },
    getAndDeleteExtractedData: function (requests) {
      if (requests.length) {
        axios.all(requests)
          .then((responses) => {
            for (let response of responses) {
              let data = response.data[0]
              axios.patch(`/api/isoqf_extracted_data/${data.id}`, { items: [] })
                .then((response) => {})
                .catch((error) => {
                  this.$emit('print-error', error)// this.printErrors(error)
                })
            }
          })
      }
    },
    parseReference: (reference, onlyAuthors = false, hasSemicolon = true) => {
      let result = ''
      const semicolon = hasSemicolon ? '; ' : ''
      if (Object.prototype.hasOwnProperty.call(reference, 'authors')) {
        if (reference.authors.length) {
          if (reference.authors.length === 1) {
            result = reference.authors[0].split(',')[0] + ' ' + reference.publication_year + semicolon
          } else if (reference.authors.length === 2) {
            result = reference.authors[0].split(',')[0] + ' & ' + reference.authors[1].split(',')[0] + ' ' + reference.publication_year + semicolon
          } else {
            result = reference.authors[0].split(',')[0] + ' et al. ' + reference.publication_year + semicolon
          }
          if (!onlyAuthors) {
            result = result + reference.title
          }
        } else {
          return 'author(s) not found'
        }
      }
      return result
    }
  }
}
</script>

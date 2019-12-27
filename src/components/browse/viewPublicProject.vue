<template>
  <b-container class="mt-4">
    <b-row
      align-h="end">
      <b-col
        sm="2"
        cols="6"
        offset-sm="10">
        <b-link :to="{ name: 'Browse' }">
          <font-awesome-icon icon="long-arrow-alt-left" title="back to Browse" /> back to Browse
        </b-link>
      </b-col>
    </b-row>
    <b-row class="mt-3">
      <b-col
        sm="6"
        cols="12">
          <h2>{{ project.name }}</h2>
          <h3>Review question</h3>
          <p>{{ project.review_question }}</p>
      </b-col>
      <b-col
        sm="6"
        cols="12">
        <h3>Authors of review</h3>
        <p>
          {{ project.authors }}
        </p>
        <h3>Has the review been published</h3>
        <p>
          {{ (project.published_status) ? 'Yes' : 'No' }}
        </p>
        <h3>Is the iSoQf being completed by the review authors?</h3>
        <p>
          {{ (project.complete_by_author) ? 'Yes' : 'No' }}
        </p>
      </b-col>
    </b-row>
    <b-row>
      <b-col
        cols="12">
        <b-table
          responsive
          :fields="tableList.fields"
          :items="lists">
          <template v-slot:cell(cerqual.option)="data">
            <template v-if="data.item.cerqual.option">
              {{ cerqual_confidence[data.item.cerqual.option].text }}
            </template>
          </template>
        </b-table>
      </b-col>
    </b-row>
  </b-container>
</template>
<script>
import axios from 'axios'

export default {
  data () {
    return {
      project: {
        authors: null,
        description: null,
        name: null,
        review_question: null
      },
      lists: [],
      tableList: {
        fields: [
          { key: 'isoqf_id', label: '#' },
          { key: 'name', label: 'Summarized review finding' },
          { key: 'cerqual.option', label: 'CERQual Assessment of confidence' },
          { key: 'cerqual.explanation', label: 'Explanation of CERQual Assessment' },
          { key: 'referencesTxt', label: 'References' }
        ]
      },
      cerqual_confidence: [
        { value: 0, text: 'High confidence' },
        { value: 1, text: 'Moderate confidence' },
        { value: 2, text: 'Low confidence' },
        { value: 3, text: 'Very low confidence' }
      ]
    }
  },
  mounted () {
    this.getPublicProject()
  },
  methods: {
    parseReference: (reference, onlyAuthors = false, hasSemicolon = true) => {
      let result = ''
      const semicolon = hasSemicolon ? '; ' : ''
      if (Object.prototype.hasOwnProperty.call(reference, 'authors')) {
        if (reference.authors.length === 1) {
          result = reference.authors[0] + ', ' + reference.publication_year + semicolon
        } else if (reference.authors.length < 3) {
          result = reference.authors[0] + ', ' + reference.authors[1] + ', ' + reference.publication_year + semicolon
        } else {
          result = reference.authors[0] + ' et al., ' + reference.publication_year + semicolon
        }
        if (!onlyAuthors) {
          result = result + reference.title
        }
        return result
      } else {
        return result
      }
    },
    getPublicProject: function () {
      axios.get(`/api/isoqf_projects/${this.$route.params.id}`)
        .then((response) => {
          const _project = response.data
          if (!_project.private) {
            this.project = _project
            this.getLists(this.project.organization, this.project.id)
          }
        })
        .catch((error) => {
          console.log(error)
        })
    },
    getLists: function (organizationId, projectId) {
      axios.get(`/api/isoqf_lists/?organization=${organizationId}&project_id=${projectId}`)
        .then((response) => {
          this.lists = response.data
          for (let list of this.lists) {
            list.evidence_profile = {}
            this.$set(list, 'referencesTxt', '')

            let findingPromise = this.getFindings(list.organization, list.id)
            findingPromise.then((result) => {
              list.evidence_profile = result.data[0].evidence_profile
            })
            for (let ref of list.references) {
              let referencePromise = this.getReference(ref)
              referencePromise.then((response) => {
                list.referencesTxt = list.referencesTxt + this.parseReference(response.data, true, true)
              })
            }
          }
        })
        .catch((error) => {
          console.log(error)
        })
    },
    getFindings: function (organizationId, listId) {
      return axios.get(`/api/isoqf_findings/?organization=${organizationId}&list_id=${listId}`)
    },
    getReference: function (refId) {
      return axios.get(`/api/isoqf_references/${refId}`)
    }
  }
}
</script>

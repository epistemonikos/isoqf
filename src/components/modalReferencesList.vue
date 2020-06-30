<template>
  <div>
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
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'modalReferencesList',
  props: {
    references: Array,
    lists: Array
  },
  mounted: function () {
    this.loadContent()
  },
  watch: {
    local_references: function () {
      this.local_references = this.references
    },
    local_lists: function () {
      this.local_lists = this.lists
    }
  },
  data: function () {
    return {
      local_references: [],
      local_lists: [],
      selected_list_index: null
    }
  },
  methods: {
    loadContent: function () {
      this.local_references = this.references
      this.local_lists = this.lists
    },
    saveReferencesList: function () {
      this.loadReferences = true
      this.table_settings.isBusy = true
      const params = {
        references: this.selected_references
      }
      axios.patch(`/api/isoqf_lists/${this.local_lists[this.selected_list_index].id}`, params)
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
    }
  }
}
</script>

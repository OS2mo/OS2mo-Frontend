SPDX-FileCopyrightText: 2018-2020 Magenta ApS SPDX-License-Identifier: MPL-2.0
<template>
  <div>
    <mo-input-date-range
      v-model="entry.validity"
      :disabled-dates="{ orgUnitValidity, disabledDates }"
    />

    <mo-organisation-unit-picker
      v-model="entry.parent"
      :label="$t('input_fields.select_super_unit')"
      :validity="entry.validity"
    />

    <div class="form-row">
      <mo-facet-picker
        v-if="showTimePlanning"
        facet="time_planning"
        v-model="entry.time_planning"
        required
      />

      <mo-facet-picker
        v-if="showOrgUnitLevel"
        facet="org_unit_level"
        v-model="entry.org_unit_level"
        required
      />
    </div>

    <div class="form-row">
      <mo-input-text :label="$t('input_fields.name')" v-model="entry.name" required />

      <mo-input-text
        v-if="showUserKey"
        :label="$t('input_fields.org_unit_user_key')"
        :placeholder="$t('input_fields.org_unit_user_key_placeholder')"
        v-model="entry.user_key"
      />

      <mo-facet-picker
        facet="org_unit_type"
        v-model="entry.org_unit_type"
        :filter_function="filter_on_owner"
        required
      />

      <mo-facet-picker
        v-if="showOrgUnitHierarchy && facetOrgUnitHierarchy.length > 1"
        facet="org_unit_hierarchy"
        v-model="entry.org_unit_hierarchy"
      />
    </div>
  </div>
</template>

<script>
/**
 * A organisation unit entry component.
 */
import MoOrganisationUnitPicker from "@/components/MoPicker/MoOrganisationUnitPicker"
import MoFacetPicker from "@/components/MoPicker/MoFacetPicker"
import { MoInputText, MoInputDateRange } from "@/components/MoInput"
import MoEntryBase from "./MoEntryBase"
import FacetOrgUnitHierarchy from "@/mixins/FacetOrgUnitHierarchy"
import { get_by_graphql } from "@/api/HttpCommon"
import moment from "moment"

export default {
  extends: MoEntryBase,
  mixins: [FacetOrgUnitHierarchy],
  name: "MoOrganisationUnitEntry",

  components: {
    MoInputDateRange,
    MoOrganisationUnitPicker,
    MoFacetPicker,
    MoInputText,
  },

  /**
   * Validator scope, sharing all errors and validation state.
   */
  inject: {
    $validator: "$validator",
  },

  props: {
    /**
     * This boolean property able the date in create organisation component.
     */
    creatingDate: Boolean,
  },

  computed: {
    globalOrgUnitLevel() {
      let conf = this.$store.getters["conf/GET_CONF_DB"]

      return conf.show_level
    },
    orgUnitValidity() {
      if (this.entry.parent) {
        return this.entry.parent.validity
      }
      return this.disabledDates
    },
    showTimePlanning() {
      if (this.entry.parent) {
        return this.entry.parent.user_settings.orgunit.show_time_planning
      } else if (this.entry.user_settings) {
        return this.entry.user_settings.orgunit.show_time_planning
      }
      return false
    },
    hasOrgUnitBeenPicked() {
      return this.entry.parent !== undefined && this.entry.parent !== null
    },
    showOrgUnitLevel() {
      if (this.hasOrgUnitBeenPicked) {
        if (this.entry.parent) {
          return this.entry.parent.user_settings.orgunit.show_level
        }
      } else if (this.entry.user_settings) {
        return this.entry.user_settings.orgunit.show_level
      }
      return this.globalOrgUnitLevel
    },
    showUserKey() {
      return !this.isEdit
    },
    getOrgUnitAncestors() {
      let entry = this.entry.parent
      if (entry === undefined) {
        return undefined
      }
      let return_val = [entry.uuid]
      while (entry.parent !== null) {
        entry = entry.parent
        return_val.push(entry.uuid)
      }
      return return_val
    },
    showOrgUnitHierarchy() {
      let conf = this.$store.getters["conf/GET_CONF_DB"]
      return conf.org_unit_hierarchy_in_create
    },
  },

  watch: {
    /**
     * Whenever orgUnit change, update newVal.
     */
    entry: {
      async handler(newVal) {
        if (newVal.parent.uuid) {
          await this.getMinMaxValidities(newVal.parent.uuid)
        }
        if (newVal.user_key === undefined || newVal.user_key === "") {
          newVal.user_key = null
        }
        this.$emit("input", newVal)
      },
      deep: true,
    },
  },

  methods: {
    getMinMaxValidities(uuid) {
      let query = `query MyQuery {
          org_units(uuids: "${uuid}", from_date: null, to_date: null){
            objects {
              validity {
                from
                to
              }
            }
          }
        }`
      get_by_graphql(query).then((response) => {
        const validities = response.data.org_units[0].objects
        let from_validities = []
        let to_validities = []
        for (let i = 0; i < validities.length; i++) {
          from_validities.push(validities[i].validity.from)
          to_validities.push(validities[i].validity.to)
        }
        var min = from_validities.reduce(function (a, b) {
          return a < b ? a : b
        })
        var max = to_validities.reduce(function (a, b) {
          return a > b ? a : b
        })
        min = min ? moment(new Date(min)).format("YYYY-MM-DD") : null
        max = max ? moment(new Date(max)).format("YYYY-MM-DD") : null
        return (this.entry.parent.validity = { from: min, to: max })
      })
    },

    filter_on_owner(classData) {
      if (classData === undefined || this.hasOrgUnitBeenPicked === false) {
        return classData
      }
      return classData.filter(
        (clazz) =>
          clazz.owner === null || this.getOrgUnitAncestors.includes(clazz.owner)
      )
    },
  },
}
</script>

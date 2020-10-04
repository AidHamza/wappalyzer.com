<template>
  <Page :title="title" :loading="loading && !error" secure>
    <v-alert v-if="success" type="success">
      {{ success }}
    </v-alert>

    <v-alert v-if="error" type="error">
      {{ error }}
    </v-alert>

    <v-card v-if="!error">
      <v-tabs
        v-model="tab"
        slider-color="primary"
        background-color="secondary"
        active-class="primary--text"
      >
        <v-tab key="manage">
          Manage
        </v-tab>
        <v-tab key="memberships">
          Memberships
        </v-tab>
      </v-tabs>

      <v-tabs-items v-model="tab">
        <v-tab-item key="manage">
          <v-card flat>
            <v-card-title>
              <v-row>
                <v-col
                  class="py-0 flex-shrink-1 flex-grow-0"
                  style="white-space: nowrap;"
                >
                  Members
                </v-col>
                <v-col class="py-0 text-right flex-shrink-0 flex-grow-1">
                  <small class="body-2 mr-4"
                    >{{ organisation.seatsRemaining }} of
                    {{ organisation.seats }} seats remaining</small
                  >
                  <v-btn
                    color="accent"
                    outlined
                    :disabled="!organisation.seatsRemaining"
                    @click="createDialog = true"
                  >
                    <v-icon left>{{ mdiAccountPlus }}</v-icon>
                    Add member
                  </v-btn>
                </v-col>
              </v-row>
            </v-card-title>
            <v-card-text class="pb-0">
              Members can create orders and spend credits.
            </v-card-text>
            <v-card-text v-if="!organisation.members.length">
              <v-alert class="ma-0" color="info" outlined>
                This organisation doesn't have any members.
              </v-alert>
            </v-card-text>
            <v-card-text v-else class="px-0">
              <v-simple-table>
                <thead>
                  <tr>
                    <th>Name</th>
                    <th>Email address</th>
                    <th>Status</th>
                    <th width="1"></th>
                  </tr>
                  <tr
                    v-for="member in organisation.members"
                    :key="member.user.sub"
                  >
                    <td>
                      <template v-if="member.user.name">
                        {{ member.user.name }}
                      </template>
                      <span v-else class="text--disabled">Not provided</span>
                    </td>
                    <td>{{ member.user.email }}</td>
                    <td>
                      <v-chip
                        v-if="member.user.sub === member.user.email"
                        color="error"
                        outlined
                        small
                        >Pending</v-chip
                      >
                      <v-chip v-else color="success" outlined small
                        >Active</v-chip
                      >
                    </td>
                    <td>
                      <v-btn
                        icon
                        @click="
                          removeUserId = member.user.sub
                          removeDialog = true
                        "
                        ><v-icon>{{ mdiCloseCircle }}</v-icon></v-btn
                      >
                    </td>
                  </tr>
                </thead>
              </v-simple-table>
            </v-card-text>
          </v-card>
        </v-tab-item>

        <v-tab-item key="memberships">
          <v-card flat>
            <v-card-title>
              Organisations
            </v-card-title>
            <v-card-text class="px-0">
              <p class="px-4">
                These are the organisations you're a member of.
              </p>

              <v-alert
                v-if="!memberOf.length"
                class="my-0 mx-4"
                color="info"
                outlined
              >
                You don't belong to any organisations.
              </v-alert>
              <v-simple-table v-else>
                <thead>
                  <tr>
                    <th>Organisation</th>
                    <th>Status</th>
                    <th width="1"></th>
                    <th width="1"></th>
                    <th width="1"></th>
                  </tr>
                  <tr
                    v-for="{ organisationId: id, name, status } in memberOf"
                    :key="id"
                  >
                    <td>
                      {{ name }}
                    </td>
                    <td>
                      <v-chip
                        v-if="status === 'Active'"
                        color="success"
                        outlined
                        small
                        >{{ status }}</v-chip
                      >
                      <v-chip v-else color="error" outlined small>{{
                        status
                      }}</v-chip>
                    </td>
                    <td>
                      <v-btn
                        v-if="status === 'Pending'"
                        color="accent"
                        outlined
                        small
                        :loading="activating"
                        @click="
                          activateOrganisationId = id
                          activate()
                        "
                        >Activate</v-btn
                      >
                    </td>
                    <td>
                      <v-btn
                        v-if="status === 'Active'"
                        color="accent"
                        outlined
                        small
                        :loading="switching"
                        @click="switchTo(id)"
                        ><v-icon left>{{ mdiLoginVariant }}</v-icon
                        >Switch to</v-btn
                      >
                    </td>
                    <td>
                      <v-btn
                        icon
                        @click="
                          removeOrganisationId = id
                          removeOrganisationDialog = true
                        "
                        ><v-icon>{{ mdiCloseCircle }}</v-icon></v-btn
                      >
                    </td>
                  </tr>
                </thead>
              </v-simple-table>
            </v-card-text>
          </v-card>
        </v-tab-item>
      </v-tabs-items>
    </v-card>

    <v-dialog v-model="createDialog" max-width="400px" eager>
      <v-card>
        <v-card-title>
          Add member
        </v-card-title>
        <v-card-text class="pb-0">
          <v-alert v-if="createError" type="error">
            {{ createError }}
          </v-alert>

          <v-form ref="form" @submit.prevent="create">
            <v-text-field
              v-model="email"
              :rules="rules.email"
              label="Email address"
              placeholder="info@example.com"
            ></v-text-field>
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="error" text @click="createDialog = false">Cancel</v-btn>
          <v-btn :loading="creating" color="accent" text @click="create"
            ><v-icon left>{{ mdiEmail }}</v-icon> Send invitation</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="removeDialog" max-width="400px" eager>
      <v-card>
        <v-card-title>
          Remove member
        </v-card-title>
        <v-card-text class="pb-0">
          <v-alert v-if="removeError" type="error">
            {{ removeError }}
          </v-alert>

          This member will be removed from the organisation.
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="accent" text @click="removeDialog = false"
            >Cancel</v-btn
          >
          <v-btn :loading="removing" color="error" text @click="remove"
            >Ok</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-dialog v-model="removeOrganisationDialog" max-width="400px" eager>
      <v-card>
        <v-card-title>
          Leave organisation
        </v-card-title>
        <v-card-text class="pb-0">
          <v-alert v-if="removeOrganisationError" type="error">
            {{ removeOrganisationError }}
          </v-alert>

          You will not be able to join to organisation again unless you're
          re-invited.
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="accent" text @click="removeOrganisationDialog = false"
            >Cancel</v-btn
          >
          <v-btn
            :loading="removingOrganisation"
            color="error"
            text
            @click="removeOrganisation"
            >Ok</v-btn
          >
        </v-card-actions>
      </v-card>
    </v-dialog>
  </Page>
</template>

<script>
import { mapState, mapActions } from 'vuex'
import {
  mdiAccountPlus,
  mdiEmail,
  mdiCloseCircle,
  mdiLoginVariant,
} from '@mdi/js'

import Page from '~/components/Page.vue'

export default {
  components: {
    Page,
  },
  data() {
    return {
      title: 'Organisation',
      createDialog: false,
      creating: false,
      createError: false,
      activating: false,
      activateOrganisationId: null,
      removeUserId: null,
      removing: false,
      removingOrganisation: false,
      removeDialog: false,
      removeOrganisationDialog: false,
      removeError: false,
      removeOrganisationError: false,
      switching: false,
      tab: 'manage',
      error: false,
      email: '',
      loading: true,
      rules: {
        email: [(v) => /@/.test(v) || 'Enter a valid email address'],
      },
      success: false,
      organisation: {
        seats: 0,
        seatsRemaining: 0,
        members: [],
      },
      mdiAccountPlus,
      mdiEmail,
      mdiCloseCircle,
      mdiLoginVariant,
    }
  },
  computed: {
    ...mapState({
      user: ({ user }) => user.attrs,
      memberOf: ({ organisations }) => organisations.memberOf,
    }),
  },
  watch: {
    async '$store.state.user.isSignedIn'(isSignedIn) {
      if (isSignedIn) {
        try {
          this.organisation = (await this.$axios.get('organisation')).data

          this.loading = false
        } catch (error) {
          this.error = this.getErrorMessage(error)
        }

        this.getMemberOf()
      }
    },
    tab(index) {
      if (index === 1) {
        if (this.$route.hash !== '#memberships') {
          this.$router.replace({ name: 'organisation', hash: '#memberships' })
        }
      } else if (this.$route.hash === '#memberships') {
        this.$router.replace({ name: 'organisation' })
      }
    },
  },
  async mounted() {
    if (this.$store.state.user.isSignedIn) {
      try {
        this.organisation = (await this.$axios.get('organisation')).data

        this.loading = false
      } catch (error) {
        this.error = this.getErrorMessage(error)
      }

      this.getMemberOf()
    }

    if (this.$route.hash === '#memberships') {
      this.tab = 1
    }
  },
  methods: {
    ...mapActions({
      getMemberOf: 'organisations/get',
    }),
    async create() {
      this.success = false
      this.error = false
      this.createError = false
      this.creating = true

      if (this.$refs.form.validate()) {
        try {
          await this.$axios.put(
            `organisation/${this.email.toLowerCase().trim()}`
          )

          this.organisation = (await this.$axios.get('organisation')).data

          this.getMemberOf()

          this.success = 'The invitation has been sent'
          this.email = ''

          this.createDialog = false
        } catch (error) {
          this.createError = this.getErrorMessage(error)
        }
      }

      this.creating = false
    },
    async remove() {
      this.success = false
      this.error = false
      this.removeError = false
      this.removing = true

      try {
        await this.$axios.delete(`organisation/${this.removeUserId}`)

        this.organisation = (await this.$axios.get('organisation')).data

        this.getMemberOf()

        this.success = 'The member has been removed.'

        this.removeDialog = false
      } catch (error) {
        this.removeError = this.getErrorMessage(error)
      }

      this.removing = false
    },
    async removeOrganisation() {
      this.success = false
      this.error = false
      this.removeOrganisationError = false
      this.removingOrganisation = true

      try {
        await this.$axios.delete(`organisation/${this.removeOrganisationId}`)

        this.organisation = (await this.$axios.get('organisation')).data

        await this.getMemberOf()

        this.success = 'You are no longer a member of the organisation.'

        this.removeOrganisationDialog = false
      } catch (error) {
        this.removeOrganisationError = this.getErrorMessage(error)
      }

      this.removingOrganisation = false
    },
    async activate() {
      this.success = false
      this.error = false
      this.activating = true

      try {
        await this.$axios.patch(`organisation/${this.activateOrganisationId}`)

        this.organisation = (await this.$axios.get('organisation')).data

        await this.getMemberOf()

        this.success = 'The organisation has been activated.'
      } catch (error) {
        this.error = this.getErrorMessage(error)
      }

      this.activating = false
    },
    async switchTo(organisationId) {
      this.error = false
      this.success = false
      this.switching = true

      this.$store.commit('user/setImpersonating', organisationId)
      this.$store.commit('user/setImpersonator', this.user)

      console.log(this.user.sub, organisationId)

      await new Promise((resolve) => {
        this.$nextTick(() => {
          try {
            // const user = (await this.$axios.get('user')).data

            // this.$store.commit('user/setAttrs', user)

            this.$router.push('/orders')
          } catch (error) {
            this.$store.commit('user/setImpersonating', '')
            this.$store.commit('user/setImpersonator', null)

            this.error = this.getErrorMessage(error)
          }

          resolve()
        })
      })

      this.switching = false
    },
  },
}
</script>
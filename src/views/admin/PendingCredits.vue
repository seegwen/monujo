<template>
  <main class="main">
    <div class="container mt-5">
      <div class="columns is-tablet">
        <div class="column">
          <div class="accounts card custom-card custom-card-padding">
            <loading
              v-model:active="isLoading"
              :can-cancel="false"
              :is-full-page="false"
            />
            <div v-if="!isLoading">
              <div
                class="notification is-danger is-light"
                v-if="hasLoadingError"
              >
                <p class="mb-4">
                  Une erreur inattendue est survenue pendant le chargement de la
                  liste de demandes de crédits. Veuillez nous excuser pour le
                  désagrément.
                </p>
                <p>
                  Vous pouvez essayer de recharger la page, ou contacter votre
                  administrateur si l'erreur persiste.
                </p>
              </div>
              <p
                class="notification is-default"
                v-else-if="pendingCreditRequests.length === 0"
              >
                Aucune demande de crédit en attente de validation
              </p>
              <div v-else>
                <h2 class="custom-card-title">
                  Opérations de crédit en attente de validation
                </h2>
                <table class="table is-striped is-fullwidth">
                  <thead>
                    <tr>
                      <th class="row-user-header">Utilisateur</th>
                      <th class="row-amount-header">Montant</th>
                      <th class="row-validate-header has-text-right">
                        Valider
                      </th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr
                      v-for="request in pendingCreditRequests"
                      v-bind:key="request"
                    >
                      <td class="row-user">
                        {{ request.relatedUser }}
                        {{
                          request.markBackend ? `(on ${request.backendId})` : ""
                        }}
                      </td>
                      <td>
                        {{
                          parseFloat(request.amount).toLocaleString("fr", {
                            minimumFractionDigits: 2,
                            maximumFractionDigits: 2,
                          })
                        }}
                        {{ request.currency }}
                      </td>
                      <td class="has-text-right">
                        <a
                          class="button is-primary custom-button custom-inverted is-small is-pulled-right"
                          v-on:click="validateCreditRequest(request)"
                          v-if="
                            selectedItem !== request || !isWaitingForValidation
                          "
                        >
                          valider
                        </a>
                        <div
                          v-else
                          class="transactions-loader-container is-pulled-right"
                        >
                          <loading
                            v-model:active="isWaitingForValidation"
                            :can-cancel="false"
                            :is-full-page="false"
                            :width="30"
                            :height="30"
                          />
                        </div>
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </main>
</template>

<script lang="ts">
  import { Options, Vue } from "vue-class-component"
  import Loading from "vue-loading-overlay"

  @Options({
    name: "PendingCredits",
    components: { Loading: Loading },
    data() {
      return {
        isLoading: true,
        hasLoadingError: false,
        isWaitingForValidation: false,
        selectedItem: null,
      }
    },
    mounted() {
      this.updatePendingCreditRequests()
    },
    computed: {
      pendingCreditRequests(): Array<any> {
        return this.$store.state.lokapi.pendingCreditRequests
      },
    },
    methods: {
      async validateCreditRequest(request: any): Promise<void> {
        this.selectedItem = request
        try {
          this.isWaitingForValidation = true
          await request.validate()
        } catch (err) {
          this.isWaitingForValidation = false
          if (err.message === "User canceled the dialog box") {
            // A warning message should have already been sent
            return
          }
          this.$msg.error(
            "Il y a eu un problème lors de la tentative de " +
              "validation de la demande de crédit" +
              request.relatedUser
          )
          throw err
        }
        this.isWaitingForValidation = false
        this.selectedItem = null
        this.$store.dispatch("fetchPendingCreditRequests")
        this.$msg.success(
          `La demande de crédit de ${request.relatedUser} ` +
          `d'un montant de ${request.amount} a été validée`,
        )
        this.isLoading = false
      },
      async updatePendingCreditRequests() {
        this.isLoading = true
        try {
          await this.$store.dispatch("fetchPendingCreditRequests")
          this.hasLoadingError = false
        } catch (e: any) {
          console.error("Failed to fetch pending credit requests", e)
          this.hasLoadingError = true
        }
        this.isLoading = false
      },
    },
  })
  export default class Admin extends Vue {}
</script>
<style scoped lang="sass">
  .row-user-header
   width: 60%
  .row-amount-header
   width: 20%
  .row-validate-header
   width: 20%
  .row-user
   overflow: hidden
   text-overflow: ellipsis
  table
   table-layout: fixed
</style>

<template>
  <div class="container-fluid">
    <h2 class="header-title mb-4">Facturadores</h2>

    <vueAlerts :alerts="alerts"></vueAlerts>
    <div class="row">
      <div class="col-12">
        <div class="card">
          <div class="card-body">
            <div class="row mb-2">
              <!-- -Link- -->
              <router-link
                tag="a"
                class="btn btn-danger waves-effect waves-light"
                :to="{ name: 'addCliente', params: { mode: 'ING' } }"
                data-animation="fadein"
                data-plugin="custommodal"
                data-overlaycolor="#38414a"
                >Agregar Facturador</router-link
              >
            </div>

            <div class="table-responsive">
              <b-table
                id="my-table"
                small
                :items="UsersRawData"
                :fields="fields"
                responsive="sm"
                :busy="tableCatIsBusy"
                class="table table-centered table-striped"
              >
                <template v-slot:table-busy>
                  <div class="text-center text-success my-2">
                    <b-spinner class="align-middle"></b-spinner>
                    <strong>Loading...</strong>
                  </div>
                </template>

                <template v-slot:cell(name)="row">
                  {{ row.item.Nom_Cli }} {{ row.item.App_Cli }}
                </template>

                <template v-slot:cell(Desc_Cat)="row">
                  <b-form-input
                    v-if="UsersRawData[row.index].actionOptions === 'editing'"
                    id="range-1"
                    v-model="row.item.Desc_Cat"
                    type="text"
                  ></b-form-input>
                  <div v-else>
                    {{ row.item.Desc_Cat }}
                  </div>
                  <!-- <input class="tabledit-input form-control form-control-sm" type="text" name="col1" value="Desc_Cat" style=""> -->
                </template>

                <template v-slot:cell(Acciones)="row">
                  <b-button-group
                    v-if="UsersRawData[row.index].actionOptions === 'default'"
                  >
                    <!--

              > -->
                    <router-link
                      tag="b-button"
                      class="btn btn-success waves-effect waves-light"
                      variant="outline-success"
                      :to="{
                        name: 'clientesDetalle',
                        params: {
                          id: UsersRawData[row.index].id,
                          mode: 'UPDATE',
                        },
                      }"
                      data-animation="fadein"
                      data-plugin="custommodal"
                      data-overlaycolor="#38414a"
                      ><span class="mdi mdi-8px mdi-pencil"></span
                    ></router-link>

                    <b-button
                      variant="outline-danger"
                      type="button"
                      @click="deleteCategory(row.index)"
                    >
                      <span class="mdi mdi-8px mdi-delete"></span>
                    </b-button>
                  </b-button-group>
                  <b-button-group
                    v-if="UsersRawData[row.index].actionOptions === 'editing'"
                  >
                    <b-button
                      variant="outline-success"
                      type="button"
                      @click="updateCategory(row.index)"
                    >
                      Actualizar
                      <span class="mdi mdi-8px mdi-content-save"></span>
                    </b-button>

                    <b-button
                      variant="outline-danger"
                      type="button"
                      @click="toggleRow(row.index)"
                    >
                      cancelar
                      <span class="mdi mdi-8px mdi-close"></span>
                    </b-button>
                  </b-button-group>

                  <!-- <button type="button" class="btn btn-outline-success btn-rounded waves-effect waves-light" style=""><span class="mdi mdi-8px mdi-pencil"></span></button> -->
                </template>
              </b-table>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- end row -->
  </div>
</template>

<script>
import vueAlerts from '../basicElements/alert'
import axios from 'axios'
import { required } from 'vuelidate/lib/validators'
import Swal from 'sweetalert2/dist/sweetalert2.js'
import 'sweetalert2/src/sweetalert2.scss' // minLength
import appConfig from '@src/app.config' // configuracion file

let catController = require('./categoriesHelpers')

export default {
  name: 'ProdInsert',
  components: {
    vueAlerts,
    // Table,Input
  },
  filters: {
    stringify(value) {
      return JSON.stringify(value, null, 2)
    },
  },
  data() {
    return {
      descCategoria: '',
      codigoCategoria: '',
      tableCatIsBusy: true,
      actionOptions: 'default',
      okAlertTimer: 0,

      alerts: [],

      fields: [
        { key: 'Dni_Cli', label: 'DNI' },
        { key: 'name', label: 'Nombre' },
        { key: 'Correo_Cli', label: 'Correo' },
        'Acciones',
      ],

      submitStatus: null,
      UsersRawData: [],

      selectedCat: null,
      productName: '',
      productDescription: '',
      dateIng: new Date(),
      dateven: new Date(),
      precioCompra: 0,
      precioVenta: 0,
      stockMin: 0,
      stockMax: 0,
      medPrescripcion: 0,

      query: '',
      selectedProveedor: null,
      proveedores: [],
    }
  },
  validations: {
    codigoCategoria: {
      required,
    },
    descCategoria: {
      required,
    },
  },
  computed: {
    provIndex: {
      get: function() {
        if (this.selectedProveedor && this.selectedProveedor.id) {
          return this.selectedProveedor.id
        } else {
          return null
        }
      },
    },
  },
  watch: {
    query(newQuery) {
      axios.get(`${appConfig}/users/?id=${newQuery}`).then((res) => {
        this.proveedores = res.data
      })
    },
  },
  // Fill Table on load
  created: function() {
    this.fillTable = () => {
      axios
        .get(`${appConfig.API_URL}clients/find`)
        .then((response) => {
          this.UsersRawData = response.data
          this.UsersRawData.forEach(function(element) {
            element.editFlag = false
            element.actionOptions = 'default'
          })
          this.tableCatIsBusy = false
        })
        .catch(function(error) {
          return error
        })
    }
    this.fillTable()
  },
  methods: {
    checkForm: function(e) {
      e.preventDefault()
      this.$v.$touch()
      if (this.$v.$invalid) {
        this.submitStatus = 'ERROR'
      } else {
        this.submitStatus = 'PENDING'
        axios
          .post('http://localhost:3010/products/insertCategory', {
            categoria: {
              Cod_Cat: this.codigoCategoria,
              Desc_Cat: this.descCategoria,
            },
          })
          .then((response) => {
            // console.dir(response)
            setTimeout((response) => {
              // console.log('ingresado')
              this.submitStatus = 'OK'
              this.fillTable()
            }, 500)
          })
          .catch((err) => {
            if (err.response && err.response.data) {
              // console.log(err.response.data)
            }
            this.submitStatus = 'ERROR'
            // console.log('error x')
            // console.log({ err })
          })
      }
    },
    toggleEdit: function(index) {
      this.UsersRawData[index].editFlag = !this.UsersRawData[index].editFlag
      this.UsersRawData[index].actionOptions = 'editing'
      this.$root.$emit('bv::refresh::table', 'my-table')
      // this.fillTable()
    },
    toggleRow: function(index, value = 'default') {
      this.UsersRawData[index].actionOptions = value
      this.$root.$emit('bv::refresh::table', 'my-table')
    },
    updateCategory: function(index) {
      catController
        .updateCategory({
          id: this.UsersRawData[index].id,
          Desc_Cat: this.UsersRawData[index].Desc_Cat,
        })
        .then(() => {
          this.UsersRawData[index].actionOptions = 'default'
          this.alerts.push({ mensaje: 'Categoria Actualizada' })
          this.$root.$emit('bv::refresh::table', 'my-table')
          // this.flash('Data loadedx', 'success');
        })
    },
    deleteCategory: function(index) {
      Swal.fire({
        title: 'Estas seguro?',
        text: 'La categoria sera eliminada!',
        icon: 'warning',
        showCancelButton: true,
        confirmButtonText: 'Si, Eliminar',
        cancelButtonText: 'No, Conservar la categoria',
      }).then((result) => {
        if (result.value) {
          Swal.fire('Eliminado!', 'Categoria Eliminada.', 'success')
          this.submitStatus = 'PENDING'
          catController
            .deleteCategory({
              id: this.UsersRawData[index].id,
            })
            .then(() => {
              this.UsersRawData[index].actionOptions = 'default'
              // this.alerts.push({mensaje: 'Categoria Eliminada'})
              this.fillTable()
              this.submitStatus = 'OK'
              this.$root.$emit('bv::refresh::table', 'my-table')
            })
        }
      })
    },

    async getAddresses(query) {
      try {
        const res = await fetch(appConfig.API_URL.replace(':query', query))
        const suggestions = await res.json()

        this.addresses = suggestions
        // console.dir(this.addresses)
      } catch (error) {
        return error // "Oops!"
      }
    },
    info(item, index, button) {
      this.infoModal.title = `Row index: ${index}`
      this.infoModal.content = JSON.stringify(item, null, 2)
      this.$root.$emit('bv::show::modal', this.infoModal.id, button)
    },
  },
}
</script>

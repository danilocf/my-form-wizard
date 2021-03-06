<template>
  <div class="my-register">

    <!-- CABECALHO -->
    <div class="my-register__header">
      <!-- LOGO -->
      <img
        v-if="formConfig.options.logoUrl"
        :src="formConfig.options.logoUrl"
        :alt="formConfig.options.title"
        class="my-register__logo"
      >
      <!-- TITULO -->
      <h1 class="my-register__title">
        {{ formConfig.options.title }}
      </h1>
    </div>

    <!-- ALERTA: ERRO AO BUSCAR CONFIGURACAO -->
    <template v-if="configError">
      <div class="my-register__error">{{ configError }}</div>
    </template>

    <!-- ALERTA: CARREGANDO CONFIGURACAO -->
    <template v-else-if="!configError && !formConfig.steps.length">
      <p>Carregando...</p>
    </template>

    <!-- ALERTA: FORMULARIO FINALIZADO -->
    <template v-else-if="submitFinished && formConfig.options.successMessage">
      <div
        class="my-register__success"
        @click.prevent="reset"
      >
        {{ formConfig.options.successMessage }}
      </div>
    </template>

    <!-- CORPO -->
    <template v-else>
      <!-- PASSOS -->
      <div
        v-for="(step, stepIndex) in formConfig.steps"
        :key="stepIndex"
      >
        <!-- PASSO ATUAL -->
        <MyForm
          v-if="activeStep === stepIndex"
          @submit="submit(stepIndex)"
        >

          <!-- INFORMACOES DO PASSO -->
          <p class="my-register__step">
            <!-- BOTAO VOLTAR -->
            <button
              v-if="showGoBackButton"
              type="button"
              class="my_form__go-back"
              @click.prevent="activeStep--"
            >
              Voltar
            </button>

            Passo: {{ stepIndex+1 }} de {{ totalSteps }}
          </p>

          <!-- TITULO DO PASSO -->
          <h2>{{ step.name }}</h2>

          <!-- CAMPOS -->
          <template v-for="item in step.items">

            <!-- SELECT -->
            <template v-if="item.type === 'select'">
              <MySelect
                :key="item.id"
                v-model="form[stepIndex][item.id]"
                :id="item.id"
                :label="item.label"
                :rules="item.rules || ''"
                :options="item.options || []"
              />
            </template>

            <!-- RADIO -->
            <template v-else-if="item.type === 'radio'">
              <MyRadio
                :key="item.id"
                v-model="form[stepIndex][item.id]"
                :id="item.id"
                :label="item.label"
                :rules="item.rules || ''"
                :options="item.options || []"
              />
            </template>

            <!-- INPUT -->
            <template v-else>
              <MyInput
                :key="item.id"
                v-model="form[stepIndex][item.id]"
                :type="item.type || 'text'"
                :id="item.id"
                :label="item.label"
                :max="item.max || 100"
                :mask="item.mask || ''"
                :rules="item.rules || ''"
              />
            </template>

          </template>

          <!-- MENSAGEM DO BOTAO DE SUBMISSAO -->
          <span slot="submit">
            {{ submitLoading ? 'Carregando...' : submitText }}
          </span>

          <!-- ALERTA - ERRO SUBMISSAO -->
          <template v-if="submitError">
            <div
              slot="error"
              class="my-register__error"
            >
              {{ submitError }}
            </div>
          </template>

        </MyForm>
      </div>
    </template>

  </div>
</template>

<script>
import Axios from 'axios'
import MyForm from '@/components/Form/MyForm'
import MyInput from '@/components/Inputs/MyInput'
import MySelect from '@/components/Inputs/MySelect'
import MyRadio from '@/components/Inputs/MyRadio'
// import mock_mega from '../mock_mega.json'

export default {
  components: {
    MyForm,
    MyInput,
    MySelect,
    MyRadio
  },
  data() {
    return {
      activeStep: 0,
      totalSteps: null,
      formConfig: { steps: [], options: { title: 'Carregando...' } }, // filled by api
      form: {}, // auto filled
      configError: null,
      submitError: null,
      submitLoading: false,
      submitFinished: false
    }
  },
  computed: {
    showGoBackButton() {
      return this.activeStep > 0
    },
    submitText() {
      if (this.activeStep+1 === this.totalSteps) return 'Finalizar'
      return 'Próximo'
    }
  },
  async created() {
    this.formConfig = await this.getConfig()
    this.generateForm()
  },
  methods: {
    async getConfig() {
      try {
        const { data } = await Axios.get('http://localhost:3000/config')
        console.log('data', JSON.stringify(data, null, '\t'))

        if (data && data.length) {
          return data[0]
        }

      } catch (error) {
        console.log('error', error)
        this.configError = 'Erro ao tentar encontrar os campos do formulário'

        return { steps: [], options: { title: 'Ops...' } }
      }
    },

    async generateForm() {
      this.form = {}

      this.formConfig.steps.forEach((step, index) => {
        this.$set(this.form, index, {})

        step.items.forEach(item => {
          const typesWithDefaultOption = ['select', 'radio']
          const model = item.id

          this.$set(this.form[index], model, null)

          if (typesWithDefaultOption.indexOf(item.type) > -1 && item.defaultOption) {
            this.form[index][model] = item.defaultOption
          }
        })
      })

      this.totalSteps = this.formConfig.steps.length
    },

    async submit(index) {
      console.log(`step ${index}`, JSON.stringify(this.form[index], null, '\t'))

      if (index+1 < this.totalSteps) {
        this.activeStep = index+1

      } else {
        const sendData = {}
        Object.keys(this.form).forEach(k => {
          Object.keys(this.form[k]).forEach(innerK => {
            sendData[innerK] = this.form[k][innerK] ? this.form[k][innerK].trim() : null
          })
        })
        sendData.project = this.formConfig.options.project
        console.log('send data', JSON.stringify(sendData, null, '\t'))

        this.submitError = ''
        this.submitLoading = true

        try {
          const { data } = await Axios.post('http://localhost:3000/register', sendData)
          console.log('data', JSON.stringify(data))

          this.submitFinished = true

        } catch (error) {
          console.log('error', error)

          if (error.response && error.response.status === 406) {
            this.submitError = 'CPF inválido'

          } else if (error.response && error.response.status === 403) {
            this.submitError = `Usuário de CPF ${sendData.cpf} já cadastrado`

          } else {
            this.submitError = 'Erro ao tentar salvar os dados'
          }

        } finally {
          this.submitLoading = false
        }
      }
    },

    reset() {
      this.activeStep = 0,
      this.totalSteps = null,
      this.formConfig = { steps: [], options: { title: 'Carregando...' } },
      this.form = {},
      this.configError = null,
      this.submitError = null,
      this.submitLoading = false,
      this.submitFinished = false
      this.generateForm()
    }
  }
}
</script>

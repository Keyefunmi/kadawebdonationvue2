<template>
  <div>
    <!--    {{this.flutterwaveRef}}-->

    <a class="button" @click="makePayment">
      <img :src="continueToForm" class="submitIconFormat">
    </a>

  </div>


</template>

<script>
import {mapFields} from "vuex-map-fields";
import {mapState} from "vuex";
import continueButoon from "@/assets/makePayment.svg";
import axios from "axios";


export default {
  name: "FlutterwaveModal",
  data: () => ({
    continueToForm: continueButoon,
    paymentResponse : '',
  }),

  created() {
    const script = document.createElement("script");
    script.src = !this.isProduction
        ? "https://ravemodal-dev.herokuapp.com/v3.js"
        : "https://checkout.flutterwave.com/v3.js";
    document.getElementsByTagName("head")[0].appendChild(script);

  },


  computed: {
    minAmountValid() {
      const val = String(this.$store.getters.minAmountValidation).replace(/,/g, "").replace("₦", "") // remove comma from the amount and naira sign
      return val

    },

    amountDonationInput: {
      get() {
        return this.$store.state.amountDonation.donationValue;
      },
      set(value) {
        this.$store.commit('SET_AMOUNT', value);
      }
    },


    // display the payment response
    responseINI() {
      return this.paymentResponse;
    },


    ...mapFields(["causeXData", "lastName", "email", "amountDonation.donationValue", "amountDonation.options", "currency", "minimumDonation", "dollarMinimumDonation", "nairaMinimumDonation", "currencyXData"])
    ,
    flutterwaveData: {
      get() {
        return this.$store.state.initFlutterData;
      }
    },
    flutterwaveRef: {
      get() {
        return this.$store.state.initFlutterData.paymentTransactionReference
      }
    },

    formIsValid: {
      get() {
        return this.$store.state.formIsValid;
      },
      set(value) {
        this.$store.commit("SET_VALID_FORM", value);
      }
    },
    emailInput: {
      get() {
        return this.$store.state.email.value
      },
      set(value) {
        this.$store.commit("SET_EMAIL", value)
      }

    },

    emailValid: {
      get() {
        return this.$store.state.email.isValid
      },
      set(value) {
        this.$store.commit("SET_EMAIL_VALID", value)
      }
    }


    , firstName: {
      get() {
        return this.$store.state.firstName.value
      },
      set(value) {
        this.$store.commit("SET_FIRST_NAME", value);
      }
    },

    firstNameValid: {
      get() {
        return this.$store.state.firstName.isValid
      },
      set(value) {
        this.$store.commit("SET_FIRST_NAME_VALID", value);
      }
    },

    lastName: {
      get() {
        return this.$store.state.lastName.value
      },
      set(value) {
        this.$store.commit("SET_LAST_NAME", value);
      }
    },


    minimumAMountNaira() {

      // check if dataresponse is not 200 return null
      if (this.causeXData.responseCode !== 200) {
        return "No data"
      }
      const amount = this.causeXData.responseContent.minimumAmountAllowed.replace(/,/g, "").replace("₦", "")
      // alert("%c amount", "color: #00ff00 ; font-size: 200px", amount)




      return amount
    },
    ...mapFields(["causeXData"]),
    ...mapState(['amountDonation']),


    donatedAmount() {
      const indoData = this.$store.state.amountDonation.donationValue
      return indoData
    },


  },


  methods: {



    validateForm() {

      this.$store.commit("SET_FIRST_NAME_VALID", true);
      this.$store.commit("SET_EMAIL_VALID", true);
      this.$store.commit("SET_LAST_NAME_VALID", true);
      this.$store.commit("SET_AMOUNT_DONATION_VALID", true);
      this.$store.commit("SET_MIN_AMOUNT_ALERT", "#003b88")

      this.formIsValid = true;


      if (this.firstName === null || this.firstName.length < 1) {
        // alert("Please enter a valid first name");

        this.formIsValid = false;
        this.$store.commit("SET_FIRST_NAME_VALID", false);
      }

      if (this.lastName === null || this.lastName.length < 1) {
        // alert("Please enter a valid last name");


        this.formIsValid = false;
        this.$store.commit("SET_LAST_NAME_VALID", false);
      }

      // validate email address using regex
      const emailRegex = /^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+@[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]+)*$/;
      if (!emailRegex.test(this.emailInput)) {
        this.formIsValid = false;
        this.$store.commit("SET_EMAIL_VALID", false);

      }

      if (this.amountDonationInput === null || this.amountDonationInput < this.minAmountValid || this.amountDonationInput.length < 1) {

        this.$store.commit("SET_AMOUNT_DONATION_VALID", false);
        this.formIsValid = false;
        this.$store.commit("SET_MIN_AMOUNT_ALERT", 'red')


      }


    }

    ,
    makePayment() {


      // this.$store.commit("SET_MIN_AMOUNT_ALERT", "#003b88")

      this.validateForm();


      if (this.formIsValid) {


        this.$store.dispatch("initializeFlutterwavePayment")


        setTimeout(() => {
          const FlutterwaveCheckout = window.FlutterwaveCheckout;
          FlutterwaveCheckout({
            public_key: this.flutterwaveData.gatewayPublicKey,
            tx_ref: this.flutterwaveData.paymentTransactionReference,
            amount: this.flutterwaveData.paymentAmount,
            currency: this.flutterwaveData.currency,
            payment_options: this.payment_method,
            customer: {
              name: this.flutterwaveData.customerName,
              email: this.flutterwaveData.emailAddress,
            },
            customizations: {
              title: this.custom_title,
              description: this.flutterwaveData.narration,
              logo: this.flutterwaveData.logo,
            },
            onclose: function () {

              // this.$store.commit("SET_AMOUNT", null)
              // this.$store.commit("SET_FIRST_NAME", null)
              // this.$store.commit("SET_LAST_NAME", null)
              // this.$store.commit("SET_EMAIL", null)


              // this.$router.push({ name: "paymentrestart" });


            },
            callback(response) {


              this.paymentResponse = response

              console.log("$$$$$$$$   ", JSON.stringify(this.paymentResponse))

              console.log(response)

              // send the response to the server

              try {

                /// send axios request to the server
                axios.post("https://kada.identity.stage.wealthtech.ng/transaction/donation/public/collection/verification", {
                  "deviceId": "string",
                  "deviceName": "string",
                  "deviceOS": "string",
                  "osVersion": "string",
                  "paymentId": response.transaction_id,

                  "paymentReference": response.tx_ref,
                  "paymentChannel": "Flutterwave",
                })
                  .then(response => {
                    console.log("%c axios response ", response.data.responseMessage, "%c color: #00ff00 ; font-size: 200px")
                    1
                    if (response.data.responseMessage === "Payment Completed!") {

                      window.location.href = '/paymentsuccess'
                    } else {
                      console.log("----------------------------------------------------")

                      // window.location.href = '/causecontribution/paymentfailure'
                      // window.location.href = window.location.hostname + "/paymentsuccess"
                      window.location.href = '/causecontribution/:id/paymentrestart'


                    }

                  })
                  .catch(error => {
                    // window.location.href = '/causecontribution/paymentfailure'
                    window.location.href = '/causecontribution/:id/paymentrestart'
                    console.log("xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx")

                    console.log(error);
                  });

              } catch (e) {

                console.log(e + "Catch $$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$")
                // window.location.href = window.location.hostname + "/causecontribution/:id/paymentrestart"
                window.location.href = '/causecontribution/:id/paymentrestart'

              }

            }


});
        }, 1000);

      }

    }



    ,

    processPaymentResponse() {
      alert("callback payment")
      console.log(this.paymentResponse)
    }

  },

  watch:{

    responseIN(){
      alert("responsex")

      const data  =  this.paymentResponse

      console.log(data)

      return data

    }


  }
      ,
  props: {
    // isProduction: {
    //   type: Boolean,
    //   required: false,
    //   default: false //set to true if you are going live
    // },
    // email: {
    //   type: String,
    //   required: true
    // },
    // amount: {
    //   type: Number,
    //   required: true
    // },
    // flwKey: {
    //   type: String,
    //   required: true
    // },
    // callback: {
    //   type: Function,
    //   required: true,
    //
    //   // eslint-disable-next-line
    //   default: (response) => {
    //
    //   }
    // },
    // close: {
    //   type: Function,
    //   required: true,
    //   default: () => {
    //   }
    // },
    // currency: {
    //   type: String,
    //   default: "NGN"
    // },
    // country: {
    //   type: String,
    //   default: "NG"
    // },
    // custom_title: {
    //   type: String,
    //   default: ""
    // },
    // custom_logo: {
    //   type: String,
    //   default: ""
    // },
    // reference: {
    //   type: String,
    //   default: ""
    // },
    // payment_method: {
    //   type: String,
    //   default: "card,mobilemoney,ussd"
    // }
  },

}
</script>

<style scoped>


img.submitIconFormat[data-v-4506583d] {
  width: 19.5rem;
  height: 3.4rem;
}


@media screen and (max-width: 375px) {


  img.submitIconFormat[data-v-4506583d] {
    width: 14.5rem;
    height: 2.4rem;
  }


}



img.submitIconFormat {
  width: 19.3rem;
}

a.button {
  display: flex;
  padding-left: 1.3rem;
  padding-top: 1.4rem;
}

@media screen and (max-width: 900px) {

  a.button[data-v-26c494ba] {
    display: flex;
    padding-left: 1.3rem;
    padding-top: 0rem;
  }

  img.submitIconFormat[data-v-4506583d] {
    width: 20.5rem;
    height: 3.4rem;
  }

  img.submitIconFormat[data-v-26c494ba] {
    width: 20.5rem;
    height: 3.4rem;
  }

}


</style>
<template>
  <div class="page-container">
    <!-- Display only the QR code image if isQrOnly is true (for emails or image embedding) -->
    <div v-if="isQrOnly" class="qr-only-container">
      <img :src="qrCodeBase64" alt="PayNow QR Code" />
    </div>

    <!-- Full payment page content if isQrOnly is false -->
    <b-card v-else class="payment-card">
      <div v-if="currency === 'SGD'" class="payment-container">
        <div class="d-flex align-items-start">
          <!-- QR Code Section -->
          <div class="qr-code-section">
            <div ref="qrcodearea" class="qr-code-wrapper"></div>
          </div>

          <!-- Payment Details and Instructions Section -->
          <div class="payment-details">
            <h2 class="payment-title">XPS Pte Ltd</h2>
            <p class="ref-number"><strong>Reference:</strong> {{ refNumber }}</p>
            <p class="amount"><strong>Amount:</strong> SGD {{ amount }}</p>

            <div class="instructions">
              <p class="instruction-text">
                Scan this QR code using your bank app to complete the payment.
                <b-badge variant="success" class="recommended">
                  <BIconCheckCircleFill /> Recommended
                </b-badge>
              </p>
              <p class="alternate-instruction">
                Or send <strong>SGD {{ amount }}</strong> to <strong>UEN {{ uen }}</strong> with reference <code>{{ refNumber }}</code>.
              </p>
            </div>

            <hr class="divider" />

            <p class="final-note">
              After completing the payment, you may close this window. We will verify your payment soon.
            </p>

            <div class="button-group">
              <b-button
                size="sm"
                variant="primary"
                @click="back"
                v-if="!noBack"
                aria-label="Back to Invoice"
                ><BIconArrowLeft /> Back to Invoice</b-button
              >
              <b-button
                size="sm"
                variant="secondary"
                class="ml-1"
                @click="close"
                aria-label="Close Window"
                ><BIconX /> Close this window</b-button
              >
            </div>
          </div>
        </div>
      </div>

      <!-- Unsupported Currency Message -->
      <div v-else class="unsupported-message">
        <p class="lead">PayNow payments are only supported for transactions in Singapore Dollars (SGD).</p>
        <p>Please use a different payment method.</p>
        <b-button size="sm" variant="primary" @click="back" v-if="!noBack">
          <BIconArrowLeft /> Back to Invoice
        </b-button>
        <b-button size="sm" variant="secondary" class="ml-1" @click="close">
          <BIconX /> Close this window.
        </b-button>
      </div>
    </b-card>
  </div>
</template>


<script>
import PaynowQR from 'paynowqr';
import paynowlogo from '@/assets/paynow.png';
import QRCode from 'easyqrcodejs';
import QRCode2 from 'qrcode';
import {
  BIconCheckCircleFill,
  BIconX,
  BIconArrowLeft
} from 'bootstrap-vue';

export default {
  name: 'paynow',
  components: {
    BIconCheckCircleFill,
    BIconX,
    BIconArrowLeft
  },
  data() {
    const { amount, refNumber, currency, noBack, output } = this.parseUrlParams();
    const uen = process.env.VUE_APP_UEN;
    const company = process.env.VUE_APP_ORGNAME;

    // Generate QR code string using PaynowQR
    const qrcode = new PaynowQR({
      uen,
      amount: amount.replace(/,|\s/g, ''),
      refNumber,
      company
    });

    return {
      qrString: qrcode.output(),
      refNumber,
      amount,
      uen,
      company,
      currency,
      noBack,
      qrCodeBase64: '', // Will hold the base64 version of QR code
      isQrOnly: output === 'qrOnly' // Determine if only QR code image should be shown
    };
  },
  methods: {
    close() {
      window.close();
    },
    back() {
      window.history.back();
    },
    parseUrlParams() {
      const urlParams = new URLSearchParams(window.location.search);
      return {
        amount: urlParams.get('amount') || urlParams.get('a'),
        refNumber: urlParams.get('invoiceNo') || urlParams.get('i'),
        currency: urlParams.get('currency') || urlParams.get('c'),
        noBack: urlParams.get('noBack') || urlParams.get('b'),
        output: urlParams.get('output') // Determines if only QR code image should be output
      };
    },
    async generateBase64QRCode() {
      try {
        this.qrCodeBase64 = await QRCode2.toDataURL(this.qrString, {
          width: 256,
          color: {
            dark: "#7D1979",  // QR code color
            light: "#ffffff"  // Background color
          }
        });
        console.log('QR Code generated successfully:', this.qrCodeBase64); // Debugging log
      } catch (error) {
        console.error('QR Code generation failed:', error);
      }
    }
  },
  mounted() {
    // Parse URL parameters in mounted to ensure they are set before rendering
    const { amount, refNumber, currency, noBack, output } = this.parseUrlParams();
    
    this.amount = amount;
    this.refNumber = refNumber;
    this.currency = currency;
    this.noBack = noBack;
    this.isQrOnly = output === 'qrOnly'; // Set isQrOnly based on output parameter
    console.log('isQrOnly:', this.isQrOnly); // Debugging to confirm isQrOnly value

    const uen = process.env.VUE_APP_UEN;
    const company = process.env.VUE_APP_ORGNAME;

    const qrcode = new PaynowQR({
      uen,
      amount: amount.replace(/,|\s/g, ''),
      refNumber,
      company
    });
    this.qrString = qrcode.output();
    console.log('QR String for QR Code generation:', this.qrString); // Debugging line

    // Conditionally generate the QR code based on isQrOnly
    if (this.currency === 'SGD') {
      if (this.isQrOnly) {
        console.log('Generating QR Code as Base64 image...');
        this.generateBase64QRCode();
        console.log('QR Code as base64:', this.qrCodeBase64); // Debugging to confirm base64 output
      } else {
        new QRCode(this.$refs.qrcodearea, {
          text: this.qrString,
          colorDark: '#7D1979',
          logo: paynowlogo,
          correctLevel: QRCode.CorrectLevel.H
        });
      }
    } else {
      console.log('Currency is not SGD, displaying alternative message.');
    }
  }
};
</script>
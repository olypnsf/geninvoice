<template>
  <div class="invoice-form">
    <h2>Invoice Details</h2>
    <form @submit.prevent="generatePDF">
      <div class="form-group">
        <label for="recipient">Recipient:</label>
        <input id="recipient" v-model="invoice.recipient" type="text" required>
      </div>
      <div class="form-group">
        <label for="date">Date:</label>
        <input id="date" v-model="invoice.date" type="date" required>
      </div>
      <div class="form-group">
        <label>Items:</label>
        <div v-for="(item, index) in invoice.items" :key="index" class="item-row">
          <input v-model="item.description" type="text" placeholder="Description" required>
          <input v-model="item.quantity" type="number" placeholder="Quantity" required>
          <input v-model="item.price" type="number" placeholder="Price" required>
          <button @click.prevent="removeItem(index)">Remove</button>
        </div>
        <button @click.prevent="addItem">Add Item</button>
      </div>
      <div class="form-group">
        <label for="imageUpload">Invoice Logo/Image:</label>
        <input type="file" id="imageUpload" @change="handleImageUpload" accept="image/*">
      </div>
      <div class="form-group">
        <label for="tax">Tax Percentage (%):</label>
        <input id="tax" v-model.number="invoice.tax" type="number" min="0" placeholder="0">
      </div>
      <button type="submit" class="submit-button">Generate Invoice</button>
    </form>

    <!-- PDF Preview Section -->
    <div v-if="pdfPreviewUrl" class="pdf-preview">
      <iframe :src="pdfPreviewUrl" width="100%" height="500px"></iframe>
      <button @click="downloadPDF">Download PDF</button>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue';
import { PDFDocument } from 'pdf-lib';

export default {
  setup() {
    const invoice = ref({
      recipient: '',
      date: '',
      items: [{ description: '', quantity: 1, price: 0 }],
      tax: 0,
      image: null,
    });

    const pdfPreviewUrl = ref(null);

    function addItem() {
      invoice.value.items.push({ description: '', quantity: 1, price: 0 });
    }

    function removeItem(index) {
      invoice.value.items.splice(index, 1);
    }

    async function handleImageUpload(event) {
      const file = event.target.files[0];
      if (file && file.type.startsWith('image/')) {
        const reader = new FileReader();
        reader.onload = function (e) {
          invoice.value.image = e.target.result;
        };
        reader.readAsDataURL(file);
      }
    }

    async function generatePDF() {
      const pdfDoc = await PDFDocument.create();
      const page = pdfDoc.addPage();
      const { width, height } = page.getSize();
      let yOffset = height - 100; // Adjust starting position for the text

      if (invoice.value.image) {
        const imageBytes = await fetch(invoice.value.image).then(res => res.arrayBuffer());
        const embeddedImage = await pdfDoc.embedPng(imageBytes);
        const imageHeight = embeddedImage.height;
        const imageWidth = embeddedImage.width;
        const imageScale = 150 / imageWidth;
        const imageHeightScaled = imageHeight * imageScale;

        page.drawImage(embeddedImage, {
          x: 50,
          y: height - 100,
          width: 150,
          height: imageHeightScaled,
        });
        yOffset -= imageHeightScaled + 20;
      }

      const fontSize = 12;

      page.drawText(`Invoice for: ${invoice.value.recipient}`, {
        x: 50,
        y: yOffset,
        size: fontSize + 8,
      });
      yOffset -= 20;

      page.drawText(`Date: ${invoice.value.date}`, {
        x: 50,
        y: yOffset,
        size: fontSize,
      });
      yOffset -= 30;

      // Items table headers
      page.drawText("Description", { x: 50, y: yOffset, size: fontSize });
      page.drawText("Quantity", { x: 200, y: yOffset, size: fontSize });
      page.drawText("Price", { x: 300, y: yOffset, size: fontSize });
      page.drawText("Total", { x: 400, y: yOffset, size: fontSize });
      yOffset -= 15;

      let subtotal = 0;
      invoice.value.items.forEach(item => {
        let itemTotal = item.quantity * item.price;
        subtotal += itemTotal;
        page.drawText(item.description, { x: 50, y: yOffset, size: fontSize });
        page.drawText(item.quantity.toString(), { x: 200, y: yOffset, size: fontSize });
        page.drawText(`$${item.price.toFixed(2)}`, { x: 300, y: yOffset, size: fontSize });
        page.drawText(`$${itemTotal.toFixed(2)}`, { x: 400, y: yOffset, size: fontSize });
        yOffset -= 15;
      });

      // Tax calculations
      let taxAmount = subtotal * (invoice.value.tax / 100);
      let total = subtotal + taxAmount;
      yOffset -= 10; // Space before totals
      page.drawText(`Subtotal: $${subtotal.toFixed(2)}`, {
        x: 400,
        y: yOffset,
        size: fontSize,
      });
      yOffset -= 15;
      page.drawText(`Tax: $${taxAmount.toFixed(2)} (${invoice.value.tax}%)`, {
        x: 400,
        y: yOffset,
        size: fontSize,
      });
      yOffset -= 15;
      page.drawText(`Total: $${total.toFixed(2)}`, {
        x: 400,
        y: yOffset,
        size: fontSize + 2,
      });

      const pdfBytes = await pdfDoc.save();
      const blob = new Blob([pdfBytes], { type: "application/pdf" });
      pdfPreviewUrl.value = URL.createObjectURL(blob);
    }

    function downloadPDF() {
      if (!pdfPreviewUrl.value) return;
      const link = document.createElement("a");
      link.href = pdfPreviewUrl.value;
      link.download = "invoice.pdf";
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
      URL.revokeObjectURL(pdfPreviewUrl.value);
      pdfPreviewUrl.value = null; // Reset preview URL
    }

    return {
      invoice,
      pdfPreviewUrl,
      addItem,
      removeItem,
      handleImageUpload,
      generatePDF,
      downloadPDF,
    };
  },
};
</script>


<style scoped>
label{
  background-color: rgb(255, 255, 255);
  font-family: 'Segoe UI',sans-serif;
}
button{
  background-color: navy;
color: white;
border: none;
border-radius: 54px;
padding: 6px 12px;
}
.invoice-form {
  max-width: 600px;
  margin: auto;
  padding: 20px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.invoice-form h2 {
  margin-bottom: 20px;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
}

.form-group input[type="text"],
.form-group input[type="number"],
.form-group input[type="date"] {
  width: 100%;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-family: 'Segoe UI',sans-serif;
}

.item-row {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}

.item-row button {
  padding: 6px 12px;
  background-color: #f44336;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.submit-button {
  padding: 10px 20px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.submit-button:hover {
  background-color: #45a049;
}

.pdf-preview iframe {
  border: 1px solid #ccc;
  border-radius: 4px;
}

.pdf-preview button {
  margin-top: 15px;
  padding: 10px 20px;
  background-color: #008CBA;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.pdf-preview button:hover {
  background-color: #007ba7;
}
#imageUpload{
  font-family: 'Segoe UI',sans-serif;
 background-color: navy;
  color: white;
}
</style>

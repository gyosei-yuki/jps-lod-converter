<template>
  <div class="converter-container">
    <h2>🏛️ 岩手県オープンデータ RDF(JSON-LD) コンバーター</h2>
    <p>岩手県の観光施設リスト（Excel/CSV）を、ジャパンサーチ標準形式に一発変換します。</p>

    <!-- ファイル読み込みボタン -->
    <div class="upload-area">
      <input type="file" @change="handleFileUpload" accept=".xlsx, .xls, .csv" />
      <button @click="convertData" :disabled="!rawData.length" class="convert-btn">
        JSON-LDに変換！
      </button>
    </div>

    <!-- 結果表示エリア -->
    <div v-if="jsonLdResult" class="result-area">
      <h3>✨ 変換完了！これがLOD（Linked Open Data）です</h3>
      <textarea v-model="jsonLdResult" rows="15" readonly></textarea>
      <button @click="downloadJsonLd" class="dl-btn">ファイルをダウンロード</button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import * as XLSX from 'xlsx' // ★追加：Excel解析ライブラリの召喚

const rawData = ref([])
const jsonLdResult = ref('')

// 1. 本物のExcelファイルを読み込んで配列（JSON）にする魔法
const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return

  // ブラウザの機能（FileReader）を使ってファイルを読み込む
  const reader = new FileReader()

  reader.onload = (e) => {
    // 読み込んだバイナリデータをXLSXライブラリに渡す
    const data = new Uint8Array(e.target.result)
    const workbook = XLSX.read(data, { type: 'array' })

    // 1つ目のシートの名前を取得して、そのシートのデータを読み込む
    const firstSheetName = workbook.SheetNames[0]
    const worksheet = workbook.Sheets[firstSheetName]

    // Excelの表を、JavaScriptの配列（JSON）に一瞬で変換！
    // defval: "" をつけることで、空っぽのセルもエラーにならず空文字として取得できます
    const jsonData = XLSX.utils.sheet_to_json(worksheet, { defval: '' })

    rawData.value = jsonData
    alert(
      `${file.name} を読み込みました！データ件数: ${jsonData.length}件\n変換ボタンを押してください。`,
    )
  }

  // 読み込み開始！
  reader.readAsArrayBuffer(file)
}

// 2. ボタンから呼ばれる変換関数（ここは先ほどと同じ完璧なロジック！）
const convertData = () => {
  const validData = rawData.value.filter((item) => item['名称'] && item['名称'] !== '◎')

  const jsonLdArray = validData.map((item) => {
    let cleanUrl = item['URL'] ? String(item['URL']) : ''
    if (cleanUrl && !cleanUrl.startsWith('http')) {
      cleanUrl = 'https://' + cleanUrl
    }

    return {
      '@context': 'http://schema.org',
      '@type': 'TouristAttraction',
      name: item['名称'],
      alternateName: item['名称_カナ'] || undefined,
      description: item['説明'] || undefined,
      url: cleanUrl || undefined,
      location: {
        '@type': 'Place',
        address: item['住所'],
        telephone: item['連絡先電話番号'] || undefined,
        geo:
          item['緯度'] && item['経度']
            ? {
                '@type': 'GeoCoordinates',
                latitude: parseFloat(item['緯度']),
                longitude: parseFloat(item['経度']),
              }
            : undefined,
      },
    }
  })

  jsonLdResult.value = JSON.stringify(jsonLdArray, null, 2)
}

// 3. ダウンロード機能
const downloadJsonLd = () => {
  const blob = new Blob([jsonLdResult.value], { type: 'application/ld+json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'iwate_tourist_jps.jsonld'
  a.click()
  URL.revokeObjectURL(url)
}
</script>

<style scoped>
.converter-container {
  padding: 30px;
  background-color: #f8f9fa;
  border-radius: 8px;
  max-width: 800px;
  margin: 0 auto;
}
.upload-area {
  margin: 20px 0;
  padding: 20px;
  border: 2px dashed #ccc;
  background: #fff;
  text-align: center;
}
.convert-btn,
.dl-btn {
  background-color: #0056b3;
  color: white;
  border: none;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
  border-radius: 5px;
  margin-top: 10px;
}
.convert-btn:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
textarea {
  width: 100%;
  font-family: monospace;
  background-color: #2d2d2d;
  color: #a8ff60;
  padding: 15px;
  border-radius: 5px;
  box-sizing: border-box;
}
</style>

<script setup>
import { ref } from 'vue';

const BASE_URL = 'http://165.22.108.142:3000/api'
const otp = ref('')
const typeOtp = ref('call')
const listDevice = ref([])
const enableCall = ref(true)

function delay(time) {
  return new Promise((res) => {
    setTimeout(res, time)
  })
}

async function activeDevice() {
  try {
    const resOtp = await fetch(
      `${BASE_URL}/auth-device/active`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          otp: otp.value,
          type: typeOtp.value
        })
      }
    )
    const resOtpJson = await resOtp.json()

    console.log(resOtpJson)

    const resInfo = await fetch(
      `${BASE_URL}/auth-device/get-info`,
      {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
          authorization: `Bearer ${resOtpJson.access_token}`
        },
      }
    )
    const resInfoJson = await resInfo.json()

    let devices = JSON.parse(localStorage.getItem('devices') || '[]')

    devices = devices.filter((device) => device._id !== resInfoJson._id)
    devices.push({ _id: resInfoJson._id, ...resOtpJson})

    localStorage.setItem('devices', JSON.stringify(devices))
  } catch (error) {
    console.log(error)
  }
}

async function getListDevice() {
  try {
    const devices = JSON.parse(localStorage.getItem('devices') || '[]')

    const res = await Promise.all(
      devices.map((device) => fetch(
        `${BASE_URL}/auth-device/get-info`,
        {
          method: 'GET',
          headers: {
            'Content-Type': 'application/json',
            authorization: `Bearer ${device.access_token}`
          },
        }
      ))
    )

    const resJson = await Promise.all(res.map((i) => i.json()))
    listDevice.value = resJson.map((i) => {
      const token = devices.find((device) => device._id === i._id)

      return {...i, ...token}
    })
  } catch (error) {
    console.log(error)
  }
}

async function startCall(device) {
  try {
    const resInfo = await fetch(
      `${BASE_URL}/phone-devices/${device._id}/number-to-call`,
      {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
          authorization: `Bearer ${device.access_token}`
        },
      }
    )
    const resInfoJson = await resInfo.json()
    const answerDevice = listDevice.value.find((device) => device.phone_number === resInfoJson.phone_number)

    console.log('----------------')
    console.log((new Date).toString())
    console.log('Gọi ---', resInfoJson)
    console.log('Id gọi ---', device._id)
    console.log('Id nghe ---', answerDevice._id)
    console.log('----------------')
    if (!resInfoJson.phone_number || !resInfoJson.duration || !resInfoJson ) throw new Error()

    setTimeout(async () => {
      try {
        console.log('----------------')
        console.log((new Date).toString())
        console.log('Tạo lich sử', {
          call_number: device.phone_number,
          answer_number: resInfoJson.phone_number,
          duration: resInfoJson.duration
        })
        console.log('Id gọi ---', device._id)
        console.log('Id nghe ---', answerDevice._id)
        console.log('----------------')

        await Promise.all([
          fetch(
            `${BASE_URL}/phone-histories/${device._id}`,
            {
              method: 'POST',
              headers: {
                'Content-Type': 'application/json',
                authorization: `Bearer ${device.access_token}`
              },
              body: JSON.stringify({
                type: 'call',
                call_number: device.phone_number,
                answer_number: resInfoJson.phone_number,
                duration: resInfoJson.duration
              })
            }
          ),
          fetch(
            `${BASE_URL}/phone-histories/${answerDevice._id}`,
            {
              method: 'POST',
              headers: {
                'Content-Type': 'application/json',
                authorization: `Bearer ${answerDevice.access_token}`
              },
              body: JSON.stringify({
                type: 'call',
                call_number: device.phone_number,
                answer_number: resInfoJson.phone_number,
                duration: resInfoJson.duration
              })
            }
          )
        ])

        console.log('delay')
        await getListDevice()
        await delay(resInfoJson.delay * 1000)
        console.log('end delay')

        if (!enableCall.value) {
          console.log('Dừng', device)
          return
        }
        startCall(device)
      } catch (error) {
        console.log(error)
      }
    }, resInfoJson.duration * 1000)
  } catch (error) {
    console.log(error)
  }
}
</script>

<template>
  <div class="container">
    <div>
      <form action="#" @submit.prevent="activeDevice">
        <input v-model="otp" type="text" placeholder="OTP">
        <select v-model="typeOtp" name="type" id="type" style="margin: 0 20px;">
          <option value="answer">Nghe</option>
          <option value="call">Gọi</option>
        </select>
        <button type="submit">Kích hoạt</button>
      </form>
    </div>
    <button style="margin: 20px 0;" @click="enableCall = false">Dừng</button>
    <div>
      <button @click="getListDevice" style="margin: 20px 0;">Lấy danh sách</button>

      <div v-for="i in listDevice" :key="i._id" style="padding: 12px 0; border-bottom: 1px solid gray;">
        <span>{{ i.type }}</span> -
        <span>{{ i._id }}</span> -
        <span>{{ i.name }}</span> -
        <span>{{ i.phone_number }}</span> -
        <span>{{ i.status }}</span>
        <button @click="startCall(i)" :disabled="i.type !== 'call'" style="margin-left: 20px;">Gọi</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  padding: 40px;
  font-family: sans-serif;
}
</style>

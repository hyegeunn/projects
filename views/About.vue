<template>
  <div class="home">
    <video id="localVideo" width="480" height="360" autoplay playsinline></video>
  </div>
  <div>
    <button @click="startWebRTC">Start</button>
    <button @click="endWebRTC">Hang Up</button>
  </div>
</template>

<script>
// @ is an alias to /src
// let localStream
let peerMyId
let peerRemoteId
const otherPeerIds = {}
let isOutRoom
let isDone
let peerConnection
let xhrRequest
let xhrHangingGet
// const receivePeerConnection = {}
const roomName = 'myclient'
const pcConfig = {
  iceServers: [
    {
      urls: [
        'stun:stun.l.google.com:19302',
        'stun:stun1.l.google.com:19302',
        'stun:stun2.l.google.com:19302',
        'stun:stun3.l.google.com:19302',
        'stun:stun4.l.google.com:19302'
      ]
    }
  ]
}
const pcOptions = {
  optional: [
    { DtlsSrtpKeyAgreement: true }
  ]
}

export default {
  name: 'Home',
  components: {},
  setup () {
    const startWebRTC = async () => {
      console.log('startWebRTC')

      xhrRequest = new XMLHttpRequest()
      xhrRequest.addEventListener('readystatechange', async (e) => {
        const { target } = e
        isDone = false

        try {
          if (target.readyState === XMLHttpRequest.DONE) {
            const { status } = target
            if (status >= 200 && status < 400) {
              const peerlist = target.responseText.split('\n')
              const arrData = peerlist[0].split(',')
              peerMyId = parseInt(arrData[1])
              console.log('peerlist', peerlist)
              console.log('peerMyId', peerMyId)

              for (let i = 1; i < peerlist.length; ++i) {
                if (peerlist[i].length > 0) {
                  const parsed = peerlist[i].split(',')
                  otherPeerIds[parseInt(parsed[1])] = parsed[0]
                  if (parsed[0].search('@') === -1) {
                    if (roomName === parsed[0]) {
                      isOutRoom = true
                    }
                  }
                }
                if (i === peerlist.length - 1) {
                  isDone = true
                }
              }
              console.log('otherPeerIds', otherPeerIds)
              console.log('isOutRoom', isOutRoom)
              console.log('isDone', isDone)
              hangingGet()
              xhrRequest = null
            }
          }
        } finally {
          if (isDone) {
            if (isOutRoom) {

            }
          }
        }
      })
      xhrRequest.open('GET', `http://192.168.1.202:8888/sign_in?${roomName}`, true)
      xhrRequest.send()
    }

    const callbackHangingGet = async (e) => {
      const { target } = e
      if (target.readyState === XMLHttpRequest.DONE) {
        const { status } = target
        if (status >= 200 && status < 400) {
          const pragma = parseInt(target.getResponseHeader('Pragma'))
          const datas = target.responseText.split('\n')
          console.log(pragma, peerMyId)
          if (pragma === peerMyId) {
            const arrData = datas[0].split(',')
            const peerId = parseInt(arrData[1])
            console.log(peerId)
            console.log('test arrData', arrData)
            console.log('test datas', datas)

            if (peerConnection) {
              if (peerId === peerRemoteId && parseInt(arrData[2]) === 0) {
                return endWebRTC()
              }
            }
          } else {
            const data = JSON.parse(target.responseText)
            console.log('pragma !== peerMyId', data)

            if (data.candidate) {
              handleCandidate(data)
            } else if (data.type === 'offer') {
              peerRemoteId = pragma
              handleOffer(peerRemoteId, data)
            } else {
              handleAnswer(data)
            }
          }
          if (xhrHangingGet) {
            xhrHangingGet.abort()
            xhrHangingGet = null
          }

          if (peerMyId !== -1) {
            setTimeout(() => {
              hangingGet()
            }, 0)
          }
        }
      }
    }

    const hangingGet = async () => {
      console.log('hangingGet')
      xhrHangingGet = new XMLHttpRequest()
      xhrHangingGet.addEventListener('readystatechange', callbackHangingGet)
      xhrHangingGet.open('GET', `http://192.168.1.202:8888/wait?peer_id=${peerMyId}`, true)
      xhrHangingGet.send()
    }

    const endWebRTC = () => {
      if (peerConnection) {
        peerConnection.close()
        peerConnection = null
      }

      if (xhrRequest) {
        xhrRequest.abort()
        xhrRequest = null
      }
      if (xhrHangingGet) {
        xhrHangingGet.abort()
        xhrHangingGet = null
      }

      xhrRequest = new XMLHttpRequest()
      xhrRequest.addEventListener('readystatechange', (e) => {
        const { target } = e
        if (target.readyState === XMLHttpRequest.DONE) {
          const { status } = target
          if (status >= 200 && status < 400) {
            peerMyId = null
            console.log('endWebRTC')
          }
        }
      })
      xhrRequest.open('GET', `http://192.168.1.202:8888/sign_out?peer_id=${peerMyId}`, false)
      xhrRequest.send()
      xhrRequest = null
    }

    const createPeerConnection = async (id) => {
      peerConnection = new RTCPeerConnection(pcConfig, pcOptions)
      peerConnection.addEventListener('icecandidate', e => {
        if (e.candidate) {
          const candidate = {
            candidate: e.candidate.candidate,
            sdpMid: e.candidate.sdpMid,
            sdpMLineIndex: e.candidate.sdpMLineIndex
          }
          sendToPeer(id, candidate)
        }
      })
      peerConnection.ontrack = e => {
        const r = document.querySelector('.home')
        const v = document.createElement('video')
        console.log(r)
        v.id = `v_${id}`
        v.width = 480
        v.height = 360
        v.srcObject = e.streams[0]
        v.autoplay = true
        r.appendChild(v)
      }
      // localStream.getTracks().forEach(track => peerConnection.addTrack(track, localStream))
    }

    const sendToPeer = (id, data) => {
      if (id === peerMyId) {
        return
      }
      console.log('sendToPeer', id, peerMyId, data)
      const dataJson = JSON.stringify(data)
      const xhr = new XMLHttpRequest()
      xhr.open('POST', `http://192.168.1.202:8888/message?peer_id=${peerMyId}&to=${id}`, true)
      xhr.setRequestHeader('Content-Type', 'text/plain')
      xhr.send(dataJson)
    }

    const handleOffer = async (id, offer) => {
      if (peerConnection) {
        console.error('existing peerconnection')
        return
      }
      await createPeerConnection(id)
      await peerConnection.setRemoteDescription(offer)

      const answer = await peerConnection.createAnswer()
      peerConnection.setLocalDescription(answer)
      sendToPeer(id, answer)
    }

    const handleAnswer = async (answer) => {
      if (!peerConnection) {
        console.error('no peerconnection')
        return
      }
      await peerConnection.setRemoteDescription(answer)
    }

    const handleCandidate = async (candidate) => {
      if (!peerConnection) {
        console.error('no peerconnection')
        return
      }
      if (!candidate.candidate) {
        await peerConnection.addIceCandidate(null)
      } else {
        await peerConnection.addIceCandidate(candidate)
      }
    }

    return {
      startWebRTC,
      endWebRTC
    }
  }
}
</script>

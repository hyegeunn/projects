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
let localStream
let peerMyId
const otherPeerIds = {}
let isOutRoom
let isDone
let senderPeerConnection
let xhrRequest
let xhrHangingGet
const receivePeerConnection = {}
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
      // isOutRoom = false
      const constraints = { audio: false, video: true }
      localStream = await navigator.mediaDevices.getUserMedia(constraints)
      const localVideo = document.getElementById('localVideo')
      localVideo.srcObject = localStream

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
              if (Object.keys(otherPeerIds).length !== 0) {
                Object.keys(otherPeerIds).forEach((v, i) => {
                  setTimeout(() => {
                    makeReceiveCall(v)
                  }, i * 1000)
                })
              }
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

    const timeoutHangingGet = (e) => {
      e.target.abort()
      if (peerMyId !== -1) {
        setTimeout(() => {
          hangingGet()
        }, 0)
      }
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

            // if (!senderPeerConnection) {
            //   console.log('senderPeerConnection 1')
            //   makeCall()
            // } else {
            //   console.log('senderPeerConnection 2')
            // }
            if (receivePeerConnection[peerId]) {
              console.log('test')
              if (parseInt(arrData[2]) === 0) {
                const v = document.getElementById(`v_${peerId}`)
                console.log(v)
                if (v) {
                  v.parentNode.removeChild(v)
                }
                receivePeerConnection[peerId].close()
                delete receivePeerConnection[peerId]
                if (Object.keys(receivePeerConnection).length === 0) {
                  return endWebRTC()
                }
              }
            } else {
              setTimeout(() => {
                makeReceiveCall(peerId)
              }, 0)
            }
          } else {
            const data = JSON.parse(target.responseText)
            console.log('pragma !== peerMyId', data)

            if (data.candidate) {
              handleCandidate(pragma, data)
            } else if (data.type === 'offer') {
              handleOffer(pragma, data)
            } else {
              handleAnswer(pragma, data)
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
      xhrHangingGet.addEventListener('timeout', timeoutHangingGet)
      xhrHangingGet.addEventListener('readystatechange', callbackHangingGet)
      xhrHangingGet.open('GET', `http://192.168.1.202:8888/wait?peer_id=${peerMyId}`, true)
      xhrHangingGet.send()
    }

    const endWebRTC = () => {
      if (senderPeerConnection) {
        senderPeerConnection.close()
        senderPeerConnection = null
      }

      for (const key in receivePeerConnection) {
        if (receivePeerConnection[key]) {
          receivePeerConnection[key].close()
          delete receivePeerConnection[key]
        }
      }

      for (const key in otherPeerIds) {
        if (otherPeerIds[key]) {
          delete otherPeerIds[key]
        }
      }

      localStream.getTracks().forEach(track => track.stop())
      localStream = null
      const localVideo = document.getElementById('localVideo')
      // const remoteVideo = document.getElementById('remoteVideo')
      localVideo.srcObject = null
      // remoteVideo.srcObject = null
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

    // const createSenderPeerConnection = async () => {
    //   senderPeerConnection = new RTCPeerConnection(pcConfig, pcOptions)
    //   senderPeerConnection.addEventListener('icecandidate', e => {
    //     if (e.candidate) {
    //       const candidate = {
    //         candidate: e.candidate.candidate,
    //         sdpMid: e.candidate.sdpMid,
    //         sdpMLineIndex: e.candidate.sdpMLineIndex
    //       }
    //       sendToPeer(peerMyId, candidate)
    //     }
    //   })
    //   // senderPeerConnection.ontrack = e => {
    //   //   const r = document.querySelector('.hello')
    //   //   const v = document.createElement('video')
    //   //   console.log(r)
    //   //   v.srcObject = e.streams[0]
    //   //   r.appendChild(v)
    //   // }
    //   localStream.getTracks().forEach(track => senderPeerConnection.addTrack(track, localStream))
    // }

    const createReceivePeerConnection = async (id) => {
      if (receivePeerConnection[id]) {
        return
      }
      const peerConnection = new RTCPeerConnection(pcConfig, pcOptions)
      peerConnection.addEventListener('icecandidate', async (e) => {
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
        // const remoteVideo = document.getElementById('remoteVideo')
        // remoteVideo.srcObject = e.streams[0]
      }
      localStream.getTracks().forEach(track => peerConnection.addTrack(track, localStream))
      receivePeerConnection[id] = peerConnection
    }

    const sendToPeer = async (id, data) => {
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

    // const makeCall = async () => {
    //   console.log('makeCall')
    //   await createSenderPeerConnection()
    //   const offer = await senderPeerConnection.createOffer()
    //   await senderPeerConnection.setLocalDescription(offer)
    //   for (const [key] of Object.entries(otherPeerIds)) {
    //     sendToPeer(key, offer)
    //   }
    // }

    const makeReceiveCall = async (id) => {
      console.log('makeReceiveCall', id)
      await createReceivePeerConnection(id)
      console.log(receivePeerConnection)
      const offer = await receivePeerConnection[id].createOffer()
      receivePeerConnection[id].setLocalDescription(offer)
      sendToPeer(id, offer)
    }

    // const handleReceiveOffer = async (id, offer) => {
    //   if (receivePeerConnection[id]) {
    //     console.error('existing peerconnection')
    //     return
    //   }

    //   await crea
    // }

    const handleOffer = async (id, offer) => {
      if (receivePeerConnection[id]) {
        console.error('existing peerconnection')
        return
      }
      console.log('handleOffer', id)
      await createReceivePeerConnection(id)
      await receivePeerConnection[id].setRemoteDescription(offer)

      const answer = await receivePeerConnection[id].createAnswer()
      receivePeerConnection[id].setLocalDescription(answer)
      sendToPeer(id, answer)
    }

    const handleAnswer = async (id, answer) => {
      if (!receivePeerConnection[id]) {
        console.error('no peerconnection')
        return
      }
      await receivePeerConnection[id].setRemoteDescription(answer)
    }

    const handleCandidate = async (id, candidate) => {
      if (!receivePeerConnection[id]) {
        console.error('no peerconnection')
        return
      }
      if (!candidate.candidate) {
        await receivePeerConnection[id].addIceCandidate(null)
      } else {
        await receivePeerConnection[id].addIceCandidate(candidate)
      }
    }

    return {
      startWebRTC,
      endWebRTC
    }
  }
}
</script>

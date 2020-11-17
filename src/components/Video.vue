<template>
  <div class="col-md-6 box">
    <div class="roomTitle">
      <span v-if="loading"> Loading... {{roomName}}</span>
      <span v-else-if="!loading && roomName"> Connected to {{roomName}}</span>
      <span v-else>Select a room to get started</span>
    </div>
    <div class="row remote_video_container">
      <div id="remoteTrack"></div>
    </div>
    <div class="spacing"></div>
    <div class="row">
      <div id="localTrack"></div>
    </div>
  </div>
</template>

<script>
  import {
    EventBus
  } from '../Event'
  import Twilio, {
    createLocalVideoTrack,
  } from 'twilio-video'
  import axios from 'axios';

  export default {
    name: 'Video',
    data() {
      return {
        loading: false,
        data: {},
        localTrack: false,
        remoteTrack: '',
        activeRoom: '',
        previewTracks: '',
        identity: '',
        roomName: null,
        videoToken: '',
      }
    },
    props: ['username'],
    created() {
      EventBus.$on('show_room', (room_name) => {
        this.createChat(room_name);
      })

      // When a user is about to transition away from this page, 
      // disconnect from the room, if joined.
      window.addEventListener('beforeunload', this.leaveRoomIfJoined);
    },
    methods: {
      // Generate access token
      async getAccessToken() {
        // const VueThis = this;
        const token = this.$route.query.token;
        const room_id = this.$route.query.room_id;
        console.log(this.$route);
        const url = `https://api.bae.sandbox.bookingsafrica.com/api/event/verify?event_id=${room_id}`;
        const options = {
          method: "GET",
          url,
          headers: { Authorization: `Bearer ${token}` },
        };
        const { data: { data: { user, event, token: videoToken }}} = await axios(options);
        console.log(user);
        console.log(event);
        console.log(videoToken);
        this.videoToken = videoToken;
        return this.videoToken;
        // return await axios.get(`http://localhost:3000/token?identity=${this.username}`);
      },

      // Trigger log events 
      dispatchLog(message) {
        EventBus.$emit('new_log', message);
      },

      //    // Attach the Tracks to the DOM.
      attachTracks(participant, container) {
        // tracks.forEach((track) => container.appendChild(track.attach()));
        participant.tracks.forEach((publication) => {
          if (publication.isSubscribed) {
            container.appendChild(publication.track.attach());
          }

          participant.on("trackSubscribed", (track) => {
            container.appendChild(track.attach());
          });
        });
      },

      //     // Attach the Participant's Tracks to the DOM.
      attachParticipantTracks(participant, container) {
        //    console.log(participant);
        // let tracks = Array.from(participant.tracks.values());
        // console.log(tracks);
        this.attachTracks(participant, container);
      },

      //    // Detach the Tracks from the DOM.
      detachTracks(track) {
        track.detach().forEach((detachedElement) => {
          detachedElement.remove();
        });
      },

      //    // Detach the Participant's Tracks from the DOM.
      detachParticipantTracks(track) {
        this.detachTracks(track);
      },

      //    // Leave Room.
      leaveRoomIfJoined() {
        if (this.activeRoom) {
          this.activeRoom.disconnect();
        }
      },

      //    // Create a new chat
      createChat(room_name) {
        this.loading = true;
        const VueThis = this;
        this.getAccessToken()
          .then(videoToken => {
            VueThis.roomName = null;
            const token = videoToken;
            console.log(token);
            let connectOptions = {
              name: room_name,
              // logLevel: 'debug',
              audio: true,
              video: {
                width: 400
              }
            };

            // before a user enters a new room,
            // disconnect the user from they joined already
            this.leaveRoomIfJoined();

            // remove any remote track when joining a new room
            document.getElementById('remoteTrack').innerHTML = "";

            Twilio.connect(token, connectOptions)
              .then(room => {
                // console.log('Successfully joined a Room: ', room);
                VueThis.dispatchLog('Successfully joined a Room: ' + room_name);

                // set active toom
                VueThis.activeRoom = room;
                VueThis.roomName = room_name;
                VueThis.loading = false;

                // Attach the Tracks of all the remote Participants.
                room.participants.forEach(participant => {
                  participant.tracks.forEach(publication => {
                    if (publication.track) {
                      document.getElementById('remoteTrack').appendChild(publication.track.attach());
                    }
                  });

                  participant.on('trackSubscribed', track => {
                    document.getElementById('remoteTrack').appendChild(track.attach());
                  });
                });

                // When a Participant joins the Room, log the event.
                room.on('participantConnected', function (participant) {
                  participant.tracks.forEach(publication => {
                    if (publication.isSubscribed) {
                      const track = publication.track;
                      document.getElementById('remoteTrack').appendChild(track.attach());
                    }
                  });

                  participant.on('trackSubscribed', track => {
                    document.getElementById('remoteTrack').appendChild(track.attach());
                  });
                });

                //   // When a Participant adds a Track, attach it to the DOM.
                //   room.on('trackAdded', function(track, participant) {
                //        VueThis.dispatchLog(participant.identity + " added track: " + track.kind);
                //        let previewContainer = document.getElementById('remoteTrack');
                //        VueThis.attachTracks(track, previewContainer);
                //    });

                // When a Participant removes a Track, detach it from the DOM.
                room.on('trackRemoved', function (track, participant) {
                  VueThis.dispatchLog(participant.identity + " removed track: " + track.kind);
                  VueThis.detachTracks(track);
                });

                // When a Participant leaves the Room, detach its Tracks.
                room.on('participantDisconnected', function (participant) {
                  VueThis.dispatchLog("Participant '" + participant.identity + "' left the room");
                  VueThis.detachParticipantTracks(participant);
                });

                // if local preview is not active, create it
                if (!VueThis.localTrack) {

                  createLocalVideoTrack().then(track => {
                    let localMediaContainer = document.getElementById('localTrack');

                    localMediaContainer.appendChild(track.attach());
                    VueThis.localTrack = true;
                    // room.localParticipant.publishTrack(localVideoTrack, {
                    // priority: "high",
                    // });

                  });
                }
              })
          })
      },
    }
  }
</script>

<style>
  .remote_video_container {
    left: 0;
    margin: 0;
    border: 1px solid rgb(124, 129, 124);
  }

  #localTrack video {
    border: 3px solid rgb(124, 129, 124);
    margin: 0px;
    max-width: 50% !important;
    background-repeat: no-repeat;
  }

  .spacing {
    padding: 20px;
    width: 100%;
  }

  .roomTitle {
    border: 1px solid rgb(124, 129, 124);
    padding: 4px;
    color: dodgerblue;
  }
</style>
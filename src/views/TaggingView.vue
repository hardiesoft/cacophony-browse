<template>
  <div>
    <h1><em>Power Tagger &trade;</em></h1>
    <div class="video-container">
      <h4 v-if="readyToTag">
        Tagging track {{ currentTrackIndex + 1 }}, Recording #{{
          currentRecording.id
        }}
      </h4>
      <h4 v-else>Loading recording</h4>
      <div class="player">
        <span
          v-if="nextTrackOrRecordingTimeout !== 0"
          class="next-track-countdown"
        >
          Next track in {{ nextTrackOrRecordingTimeout }}
        </span>
        <ThermalVideoPlayer
          v-if="fileSource || loading"
          ref="thermalPlayer"
          :video-url="fileSource || ''"
          :tracks="orderedTracks"
          :current-track="currentTrackIndex"
          :can-select-tracks="false"
          :loop-selected-track="true"
          :colours="colours"
          :show-motion-paths="false"
          @request-next-recording="nextRecording"
          @ready-to-play="readyToPlay = true"
        />
        <div v-else>Loading recording...</div>
      </div>
      <b-button :disabled="!readyToTag" @click="markTrackAsSkipped"
        >Skip</b-button
      >
      <b-button :disabled="history.length === 0" @click="undo">Undo</b-button>
      <div v-if="currentRecording">
        <div class="tag-buttons">
          <b-button
            :disabled="
              !readyToTag || (currentTrackIsAlreadyTagged && !isTagged(value))
            "
            :class="{ selected: isTagged(value) }"
            :key="`${text}_${isTagged(value) ? '_tagged' : ''}`"
            v-for="{ text, value } in tags"
            @click="() => addOrRemoveTag(value)"
            >{{ text }}</b-button
          >
        </div>

        <div v-if="false">
          <h4 v-if="taggedRecordings.length !== 0">
            Tagging history
          </h4>
          <div
            v-for="{ recording, tracks: prevTracks } in taggedRecordings"
            :key="`recording_${recording.id}`"
            class="recording-history"
          >
            <p>Recording #{{ recording.id }}</p>
            <div v-for="track in prevTracks" :key="`track_${track.id}`">
              <span
                v-if="trackIsAlreadyTaggedByCurrentUser(track)"
                class="track-history"
              >
                Track {{ track.trackIndex + 1 }}:
              </span>
              <span
                v-for="tag in tagsByThisUser(track.TrackTags)"
                :key="`tag_${tag.what}`"
              >
                <b-button
                  @click="() => deleteTag(tag, recording, track)"
                  v-if="tag.what !== 'skipped'"
                >
                  <span>{{ tag.what }}</span>
                  <font-awesome-icon icon="trash" />
                </b-button>
                <span v-else>Skipped</span>
              </span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import config from "../config";
import ThermalVideoPlayer from "../components/Video/ThermalVideoPlayer.vue";
import api from "../api";
import DefaultLabels from "../const";
import Vue from "vue";
import {
  DeviceId,
  Recording,
  RecordingId,
  RecordingInfo,
  RecordingType,
  Tag,
  TagMode,
  Track,
  TrackId,
  TrackTag,
  User
} from "../api/Recording.api";

interface TaggingViewData {
  colours: string[];
  tags: { text: string; value: string }[];
  recordingsByDevice: Record<DeviceId, RecordingInfo>;
  currentDeviceId: DeviceId;
  currentTrackIndex: number;
  previousRecordingsAndTracks: {
    recordings: RecordingInfo[];
    tracks: Track[];
  }[];
  currentRecording: RecordingInfo | null;
  currentRecordingData: Recording | null;
  loading: boolean;
  taggingPending: boolean;
  readyToPlay: boolean;
  tracks: Track[];
  history: {
    tracks: Track[];
    trackIndex: number;
    recording: RecordingInfo;
    recordingData: Recording;
    tag: TrackTag;
  }[];
  nextTrackOrRecordingTimeout: number;
}

export default Vue.extend({
  name: "TaggingView",
  components: { ThermalVideoPlayer },
  data(): TaggingViewData {
    return {
      colours: [
        "yellow",
        "orange",
        "red",
        "purple",
        "lightblue",
        "limegreen",
        "black",
        "white"
      ],
      tags: [
        ...DefaultLabels.quickTagLabels().map(x => ({ text: x, value: x })),
        ...DefaultLabels.otherTagLabels()
      ],
      recordingsByDevice: {},
      currentDeviceId: -1,
      currentTrackIndex: 0,
      previousRecordingsAndTracks: [],
      currentRecording: null,
      currentRecordingData: null,
      loading: true,
      taggingPending: false,
      readyToPlay: false,
      tracks: [],
      history: [],
      nextTrackOrRecordingTimeout: 0
    };
  },
  async created() {
    const gotRecordings = await this.getRecordings();
    if (gotRecordings) {
      this.removeAllMyTags();
      // Pick a random device to begin with:
      this.pickDevice();
      this.loading = true;
      await this.pickRecording();
      this.readyToPlay = false;
      this.loading = false;
    }
  },
  methods: {
    tagsByThisUser(tags: TrackTag[]): TrackTag[] {
      const thisUser = this.currentUser.username;
      const thisUserId = this.currentUser.id;
      return tags.filter(
        (tag: TrackTag) =>
          (tag.User && tag.User.username === thisUser) ||
          tag.UserId === thisUserId
      );
    },
    async undo() {
      const {
        recording,
        recordingData,
        tracks,
        trackIndex,
        tag
      } = this.history.pop();

      await this.deleteTag(tag, recordingData.recording, tracks[trackIndex]);

      this.currentRecording = recording;
      this.currentRecordingData = recordingData;
      this.tracks = tracks;
      this.currentTrackIndex = trackIndex;
    },
    isTagged(tagValue: string): boolean {
      if (this.currentRecording && this.tracks && this.currentTrack) {
        for (const tag of this.currentTrack.TrackTags) {
          if (tag.User !== null && tag.what === tagValue) {
            return true;
          }
        }
      }
      return false;
    },
    markTrackAsSkipped() {
      const synthesisedTag = {
        id: -1,
        TrackId: this.currentTrack.id,
        what: "skipped",
        createdAt: new Date().toUTCString(),
        User: { username: this.currentUser.username, id: this.currentUser.id }
      };
      this.currentTrack.TrackTags.push(synthesisedTag);

      this.history.push({
        trackIndex: this.currentTrackIndex,
        tracks: this.orderedTracks,
        recording: this.currentRecording,
        recordingData: this.currentRecordingData,
        tag: synthesisedTag
      });

      this.nextTrackOrRecording();
    },
    nextTrackOrRecording() {
      if (this.allTracksInRecordingAreTagged) {
        this.nextRecording();
      } else {
        this.nextUntaggedTrack();
      }
    },
    nextUntaggedTrack() {
      while (
        this.trackIsAlreadyTagged(this.currentTrack) &&
        this.currentTrackIndex < this.tracks.length
      ) {
        this.currentTrackIndex++;
        console.log("advanced to track", this.currentTrackIndex);
      }
    },
    async nextRecording() {
      const prevRecording = this.currentRecording;
      this.currentRecording = null;
      const prevTracks = [...this.orderedTracks].reverse();
      this.tracks = null;
      this.previousRecordingsAndTracks.push({
        recording: prevRecording,
        tracks: prevTracks
      });
      this.loading = true;
      await this.pickRecording();
      this.readyToPlay = false;
      this.loading = false;
    },
    addOrRemoveTag(tagLabel: string) {
      if (!this.taggingPending) {
        if (this.isTagged(tagLabel)) {
          const track: Track = this.currentTrack;
          const tag = track.TrackTags.find(
            trackTags => trackTags.what === tagLabel
          );
          if (tag) {
            this.deleteTag(tag, this.currentRecordingData.recording, track);
          }
        } else {
          this.addTag(tagLabel);
        }
      }
    },
    async addTag(tagLabel: string) {
      const recordingId = this.currentRecording.id;
      const trackId = this.orderedTracks[this.currentTrackIndex].id;
      const tag = { what: tagLabel, confidence: 0.85 };
      this.taggingPending = true;
      const { result } = await api.recording.addTrackTag(
        tag,
        recordingId,
        trackId
      );
      this.taggingPending = false;
      if (result.success) {
        // Add the track tag to this.data.tracks:
        const targetTrack: Track | undefined = this.tracks.find(
          track => track.id === trackId
        );
        if (targetTrack) {
          const synthesisedTag = {
            ...tag,
            User: {
              username: this.currentUser.username,
              id: this.currentUser.id
            },
            id: result.trackTagId,
            TrackId: trackId,
            createdAt: new Date().toUTCString()
          };
          targetTrack.TrackTags.push(synthesisedTag);

          this.history.push({
            trackIndex: this.currentTrackIndex,
            tracks: this.orderedTracks,
            recording: this.currentRecording,
            recordingData: this.currentRecordingData,
            tag: synthesisedTag
          });
        }

        console.log(this.nextTrackOrRecordingTimeout);
        this.primeNextTrack(3);
      }
    },
    primeNextTrack(tillNext: number) {
      this.nextTrackOrRecordingTimeout = tillNext;
      setTimeout(() => {
        if (this.nextTrackOrRecordingTimeout !== 0) {
          this.primeNextTrack(tillNext - 1);
        } else {
          this.nextTrackOrRecording();
        }
      }, 500);
    },
    async deleteTag(tag: TrackTag, recording: RecordingInfo, track: Track) {
      // NOTE(jon): If we got the tag from the aggregate recordings query, it doesn't have a tag id, so we need to
      //  fetch that first.
      if (tag.what !== "skipped") {
        const tagClone = { ...tag };
        if (!tagClone.hasOwnProperty("id")) {
          const thisUser = this.currentUser.username;
          const { success, result } = await api.recording.tracks(recording.id);
          if (success) {
            const targetTrack = result.tracks.find(
              (trackItem: Track) => trackItem.id === track.id
            );
            if (targetTrack) {
              const targetTag = targetTrack.TrackTags.find(
                (trackTag: TrackTag) =>
                  trackTag.what === tag.what &&
                  (trackTag.User && trackTag.User.username === thisUser)
              );
              if (targetTag) {
                tagClone.id = targetTag.id;
              }
            }
          }
        }
        if (!tag.hasOwnProperty("TrackId")) {
          tagClone.TrackId = track.id;
        }
        this.taggingPending = true;
        const { success, result } = await api.recording.deleteTrackTag(
          tagClone,
          recording.id
        );
        this.taggingPending = false;

        if (success) {
          console.log("deleted tag", tagClone);
          // eslint-disable-next-line require-atomic-updates
          track.TrackTags = track.TrackTags.filter(item => item !== tag);
        }
      } else {
        // Just remove skipped tag
        track.TrackTags = track.TrackTags.filter(item => item !== tag);
      }
    },
    pickDevice() {
      const devices = Object.keys(this.recordingsByDevice).map(Number);
      // NOTE(jon): Just pick the first device, we'll rely on the backend to choose these for us.
      this.currentDeviceId = devices[0];
    },
    async pickRecording() {
      const recordings = this.recordingsByDevice[this.currentDeviceId];
      if (recordings && recordings.length !== 0) {
        // NOTE(jon): Just take the first recording, we'll rely on the backend to choose these for us.
        this.currentRecording = recordings.splice(0, 1)[0];
        // If we took the last recording for a device, remove the device,
        // and pick a next device.
        if (recordings.length === 0) {
          delete this.recordingsByDevice[this.currentDeviceId];
          this.recordingsByDevice = { ...this.recordingsByDevice };
          this.pickDevice();
        }

        const {
          tracks,
          recording
        }: {
          tracks: Track[];
          recording: Recording;
        } = await this.$store.dispatch(
          "Video/GET_RECORDING",
          this.currentRecording.id
        );
        if (!tracks || tracks.length === 0) {
          console.warn(
            "Error, no tracks for recording",
            recording.recording.id
          );
          this.pickRecording();
        } else {
          this.tracks = tracks;
          this.currentRecordingData = recording;

          // Get the first track without human tags
          this.currentTrackIndex = this.orderedTracks.findIndex(
            (track: Track) => {
              return !this.trackIsAlreadyTagged(track);
            }
          );
          console.assert(this.currentTrackIndex !== -1);
        }
      } else {
        this.pickDevice();
      }
    },

    async removeAllMyTags() {
      const query = {
        where: {
          duration: { $gte: 0 },
          type: "thermalRaw" as RecordingType
        },
        limit: 100000,
        offset: 0,
        tagMode: "tagged" as TagMode
      };
      const stringifiedQuery = {};
      for (const [key, val] of Object.entries(query)) {
        if (typeof val === "object") {
          stringifiedQuery[key] = JSON.stringify(val);
        } else {
          stringifiedQuery[key] = val;
        }
      }

      const { result, success } = await api.recording.query(stringifiedQuery);
      const thisUserId = this.currentUser.id;
      const rows = result.rows.filter((recording: RecordingInfo) => {
        return recording.Tracks.find((track: Track) => {
          return track.TrackTags.find(
            (tag: TrackTag) => tag.UserId && tag.UserId === thisUserId
          );
        });
      });
      const allTagPromises = [];
      for (const recording of rows) {
        for (const track of recording.Tracks) {
          const tags = track.TrackTags.filter(
            trackTag => trackTag.UserId === this.currentUser.id
          );
          for (const tag of tags) {
            console.log("deleting tag", tag);
            allTagPromises.push(this.deleteTag(tag, recording, track));
          }
        }
      }
      Promise.all(allTagPromises).then(() => {
        console.log("deleted all tags for", this.currentUser.username);
      });
    },
    async getRecordings(): Promise<boolean> {
      const query = {
        where: {
          duration: { $gte: 0 },
          type: "thermalRaw" as RecordingType
        },
        limit: 1000,
        offset: 0,
        tagMode: "no-human" as TagMode
      };

      const stringifiedQuery = {};
      for (const [key, val] of Object.entries(query)) {
        if (typeof val === "object") {
          stringifiedQuery[key] = JSON.stringify(val);
        } else {
          stringifiedQuery[key] = val;
        }
      }

      const { result, success } = await api.recording.query(stringifiedQuery);

      if (success) {
        // const thisUserId = this.currentUser.id;
        // const rows = result.rows.filter((recording: RecordingInfo) => {
        //   return recording.Tracks.find((track: Track) => {
        //     return track.TrackTags.find(
        //       (tag: TrackTag) => tag.UserId && tag.UserId === thisUserId
        //     );
        //   });
        // });
        // Sort by device:
        for (const row of result.rows) {
          this.recordingsByDevice[row.DeviceId] =
            this.recordingsByDevice[row.DeviceId] || [];
          this.recordingsByDevice[row.DeviceId].push(row);
        }
      }
      return success;
    },
    trackIsAlreadyTagged(track: Track): boolean {
      return (
        track.TrackTags.filter(
          (tag: TrackTag) => tag.User !== null && tag.User !== undefined
        ).length !== 0
      );
    },
    trackIsAlreadyTaggedByCurrentUser(track: Track): boolean {
      const currentUser = this.currentUser;
      return (
        track.TrackTags.filter(
          (tag: TrackTag) =>
            (tag.User && tag.User.username === currentUser.username) ||
            tag.UserId === currentUser.id
        ).length !== 0
      );
    }
  },
  computed: {
    taggedRecordings(): Recording[] {
      if (this.currentRecording && this.tracks) {
        return [
          ...this.previousRecordingsAndTracks,
          {
            recording: this.currentRecording,
            tracks: [...this.orderedTracks].reverse()
          }
        ].reverse();
      }
      return [...this.previousRecordingsAndTracks].reverse();
    },
    allTracksInRecordingAreTagged(): boolean {
      return (
        this.tracks.reduce((acc: Track[], track: Track) => {
          if (
            track.TrackTags.filter(
              (tag: TrackTag) => tag.User !== null && tag.User !== undefined
            ).length !== 0
          ) {
            acc.push(track);
          }
          return acc;
        }, []).length === this.tracks.length
      );
    },
    currentTrackIsAlreadyTagged(): boolean {
      return this.currentTrack && this.trackIsAlreadyTagged(this.currentTrack);
    },
    readyToTag(): boolean {
      return !this.loading && this.readyToPlay;
    },
    currentTrack(): Track {
      if (this.currentTrackIndex < this.tracks.length) {
        return this.orderedTracks[this.currentTrackIndex];
      }
      // NOTE: This is technically an error, but we don't really want to return undefined from this method
      return this.orderedTracks && this.orderedTracks[0];
    },
    orderedTracks(): Track[] {
      return (
        (this.tracks &&
          [...this.tracks].sort((a, b) => a.data.start_s - b.data.start_s)) ||
        []
      );
    },
    currentUser(): User {
      return this.$store.state.User.userData;
    },
    fileSource(): string | false {
      if (this.currentRecordingData) {
        return `${config.api}/api/v1/signedUrl?jwt=${this.currentRecordingData.downloadFileJWT}`;
      }
      return false;
    }
  }
});
</script>

<style scoped lang="scss">
.video-container {
  width: 640px;
  max-width: 640px;
}
.player {
  position: relative;
}
.next-track-countdown {
  position: absolute;
  top: 230px;
  z-index: 10000;
  width: 100%;
  color: white;
  font-size: 40px;
  text-align: center;
}
.btn.selected {
  background: orange;
}
</style>

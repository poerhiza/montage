<template>
  <md-app md-waterfall md-mode="fixed" :md-scrollbar="showScrollbar">
    <md-app-toolbar class="md-primary">
      <router-link to="/galleries">
        <md-button class="md-fab md-primary md-icon-button" md-menu-trigger :disabled="galleriesPageActive">
          <md-icon>home</md-icon>
          <md-tooltip md-direction="bottom">Back to galleries</md-tooltip>
        </md-button>
      </router-link>
<!-- Import Images Menu Start -->
      <md-button class="md-icon-button" md-menu-trigger @click="importImages()">
        <md-icon>add_a_photo</md-icon>
        <md-tooltip md-direction="bottom">Import Images</md-tooltip>
      </md-button>
<!-- Import Images Menu End -->
<!-- Gallery Menu Start -->
      <md-menu md-size="big" md-direction="bottom-end">
        <md-button class="md-icon-button" md-menu-trigger>
          <md-icon>gradient</md-icon>
          <md-tooltip md-direction="bottom">Manage Gallery</md-tooltip>
        </md-button>
        <md-menu-content>
          <span style="margin-left:10px;" class="md-title">Gallery Actions</span>
          <md-menu-item @click="createGallery()">
            <md-icon>note_add</md-icon>
            <span>New</span>
            <md-tooltip md-direction="right">Create new gallery</md-tooltip>
          </md-menu-item>
          <md-menu-item @click="renameGallery()" :disabled="galleriesPageActive">
            <md-icon>mode_edit</md-icon>
            <span>Edit</span>
            <md-tooltip v-bind:class="{ disabled: galleriesPageActive }" md-direction="right">Edit Gallery</md-tooltip>
          </md-menu-item>
          <md-menu-item @click="deleteGallery()" :disabled="galleriesPageActive">
            <md-icon>delete</md-icon>
            <span>Delete</span>
            <md-tooltip v-bind:class="{ disabled: galleriesPageActive }" md-direction="right">Delete Gallery</md-tooltip>
          </md-menu-item>
        </md-menu-content>
      </md-menu>
<!-- Gallery Menu End -->
<!-- Export Menu Start -->
      <md-menu
        md-size="big"
        md-direction="bottom-end"
      >
        <md-button class="md-icon-button" md-menu-trigger :disabled="galleriesPageActive">
          <md-icon>file_download</md-icon>
          <md-tooltip md-direction="bottom">Export Gallery</md-tooltip>
        </md-button>
        <md-menu-content>
          <span style="margin-left:10px;" class="md-title">Export To</span>
          <md-menu-item @click="exportGallery(gallery)">
            <md-icon>computer</md-icon>
            <span>Save to disk</span>
            <md-tooltip md-direction="right">Save gallery to your harddrive</md-tooltip>
          </md-menu-item>
          <md-menu-item @click="notImplementedYet()">
            <md-icon>storage</md-icon>
            <span>S3</span>
            <md-tooltip md-direction="right">Publish gallery to Amazon S3 storage</md-tooltip>
          </md-menu-item>
          <md-menu-item @click="notImplementedYet()">
            <md-icon>cloud</md-icon>
            <span>IPFS</span>
            <md-tooltip  md-direction="right">Publish gallery to IPFS</md-tooltip>
          </md-menu-item>
        </md-menu-content>
      </md-menu>
<!-- Export Menu Start -->
      <span class="md-title" v-bind:class="{ disabled: galleriesPageActive }">{{ gallery.title }}</span>
      <span class="flex"></span>
      <md-button class="md-icon-button" @click="maximizeApp()" v-if="!maximized">
        <md-icon>settings_overscan</md-icon>
        <md-tooltip md-direction="bottom">Maximize Window</md-tooltip>
      </md-button>
      <md-button class="md-icon-button" @click="windowApp()" v-if="maximized">
        <md-icon>crop_square</md-icon>
        <md-tooltip md-direction="bottom">Un-Maximize Window</md-tooltip>
      </md-button>
      <md-button class="md-icon-button" @click="closeApp()">
        <md-icon>close</md-icon>
        <md-tooltip md-direction="bottom">Exit Montage</md-tooltip>
      </md-button>
    </md-app-toolbar>
    <md-progress-bar v-if="imagesImported > 0" style="width:100%;" md-mode="determinate" :md-value="progressImportingImages"></md-progress-bar>
    <md-app-content id="appContentCore">
      <router-view></router-view>
      <create-gallery-dialog></create-gallery-dialog>
      <show-image-dialog></show-image-dialog>
      <import-images-dialog></import-images-dialog>
    </md-app-content>
  </md-app>
</template>

<script>
  import { join, basename } from 'path';
  import { mapGetters, mapActions } from 'vuex';

  import { EventBus } from '../store/EventBus';
  import { checksumFile, getThumbnailBlob, getImageMetadata } from '../../utils';

  import CreateGalleryDialog from './CreateGalleryDialog.vue';
  import ShowImageDialog from './ShowImageDialog.vue';
  import ImportImagesDialog from './ImportImagesDialog.vue';

  const { ipcRenderer } = require('electron');//eslint-disable-line
  const logger = require('electron-log');

  async function addImageFromFilepath(data) {//eslint-disable-line
    const reader = new FileReader();
    reader.onloadend = async function() {//eslint-disable-line
      let image = {//eslint-disable-line
        src: data.filepath,
        type: 0,
        alt: basename(data.filepath),
        thumbnail: reader.result,
        hasBeenExported: 0,
        hash: data.checksum
      };
      let metadata = false;

      if (data.opts.importExif) {
        metadata = await getImageMetadata(data.filepath);
      }

      EventBus.$emit(
        'image-added',
        {
          metadata,
          image,
          fileCount: data.fileCount,
          galleryId: data.opts.galleryId
        }
      );
    };
    const thumbnailBlob = await getThumbnailBlob(data.filepath);
    reader.readAsDataURL(thumbnailBlob);
  }

  ipcRenderer.on('images-added', async function(evt, data) {//eslint-disable-line
    addImageFromFilepath(data);
  });

  ipcRenderer.on('add-gallery', async function(evt, data) {//eslint-disable-line
    EventBus.$emit('add-gallery', data);
  });

  async function addImageFromDragDrop(e) {
    e.preventDefault();
    e.stopPropagation();
    const filesToAdd = [];
    const checksums = {};

    for (const f of e.dataTransfer.files) {//eslint-disable-line
      const supportedFileTypes = /(\.jpg|\.jpeg|\.png)$/i;

      if (supportedFileTypes.test(f.path)) {
        const checksum = await checksumFile('md5', f.path);//eslint-disable-line

        if (!checksums[checksum]) {
          checksums[checksum] = true;
          filesToAdd.push({ filepath: f.path, checksum });
        }
      }
    }
    const fileCount = filesToAdd.length;
    filesToAdd.forEach((item) => {
      item.fileCount = fileCount;
      addImageFromFilepath(item);
    });
  }

  export default {
    name: 'navigation-bar',
    components: {
      CreateGalleryDialog,
      ShowImageDialog,
      ImportImagesDialog
    },
    data: () => ({
      showScrollbar: false,
      progressImportingImages: 0,
      imagesImported: 0,
      gallery: [],
      maximized: true
    }),
    computed: {
      ...mapGetters([
        'ImageGallery/addImage',
        'ImageGallery/gallery'
      ]),
      ...mapActions([
        'ImageGallery/addGallery',
        'ImageGallery/updateGallery',
        'ImageGallery/addImageMetadata',
        'ImageGallery/loadGallery'
      ]),
      galleriesPageActive: function () {//eslint-disable-line
        return (this.$route.name === 'galleries') ? 'disabled' : false;
      }
    },
    methods: {
      closeApp() {
        ipcRenderer.send('close');
      },
      maximizeApp() {
        ipcRenderer.send('window-all-maximize');
        this.maximized = true;
      },
      windowApp() {
        ipcRenderer.send('window-all-unmaximize');
        this.maximized = false;
      },
      importImages() {
        EventBus.$emit('import-images');
      },
      createGallery() {
        EventBus.$emit('create-gallery');
      },
      renameGallery() {
        EventBus.$emit('create-gallery', this.gallery);
      },
      deleteGallery() {
        EventBus.$emit('delete-gallery', this.gallery);
      },
      async exportGallery(gallery) {
        try {
          await this.$store.dispatch('ImageGallery/loadGallery', parseInt(gallery.id, 10));
          const paths = this.gallery.images.map(imageItem => imageItem.src);
          ipcRenderer.send('export-gallery-dialog', { paths, title: this.gallery.title });
        } catch (e) {
          new Notification(//eslint-disable-line
            'Error',
            {
              body: 'Unable to export this gallery...'
            }
          );
          logger.error(e);
        }
      },
      notImplementedYet: () => {
        alert('this feature is not yet implemented...');//eslint-disable-line
      },
      updateGallery(gallery) {
        this.gallery = gallery;
      },
      async setGalleryThumbnail(thumbnail) {
        try {
          await this.$store.dispatch(
            'ImageGallery/updateGallery',
            {
              id: this.gallery.id,
              thumbnail
            }
          );
          new Notification(//eslint-disable-line
            'Success',
            {
              body: 'gallery thumbnail updated!',
              icon: join(__static, 'check.png')
            }
          );
        } catch (e) {
          new Notification(//eslint-disable-line
            'Error',
            {
              body: 'unable to update gallery thumbnail...'
            }
          );
        }
      },
      async addGallery(data) {
        data.opts.galleryId = await this.$store.dispatch(
          'ImageGallery/addGallery',
          {
            title: data.gallery.title,
            tags: ['automatic']
          }
        );
        data.gallery.files.forEach((item) => {
          item.fileCount = data.gallery.files.length;
          item.opts = data.opts;
          addImageFromFilepath(item);
        });
      },
      async imageAdded(data) {
        await this.$store.dispatch(
          'ImageGallery/addImage',
          {
            galleryId: parseInt(data.galleryId, 10),
            image: data.image
          }
        );

        if (data.metadata) {
          data.metadata.hash = data.image.hash;

          await this.$store.dispatch(
            'ImageGallery/addOrUpdateImageMetadata',
            data.metadata
          );
        }

        this.progressImportingImages = (((this.imagesImported++) / data.fileCount ) * 100);//eslint-disable-line

        if (data.fileCount <= this.imagesImported) {
          this.imagesImported = 0;
        }
      }
    },
    mounted() {
      EventBus.$on('add-gallery', this.addGallery);
      EventBus.$on('image-added', this.imageAdded);
      EventBus.$on('view-gallery', this.updateGallery);
      EventBus.$on('export-gallery', this.exportGallery);
      EventBus.$on('set-gallery-thumbnail', this.setGalleryThumbnail);

      const appCore = document.getElementById('appContentCore');

      appCore.addEventListener('drop', addImageFromDragDrop);
      appCore.addEventListener('dragover', function (e) {//eslint-disable-line
        e.preventDefault();
        e.stopPropagation();
      });
    },
    beforeDestroy() {
      EventBus.$off('add-gallery', this.addGallery);
      EventBus.$off('image-added', this.imageAdded);
      EventBus.$off('view-gallery', this.updateGallery);
      EventBus.$off('export-gallery', this.exportGallery);
      EventBus.$off('set-gallery-thumbnail', this.setGalleryThumbnail);
    }
  };
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Roboto:300,400,500,700,400italic');

  body { font-family: 'Roboto', sans-serif; background-color: #070e1a;}

  .md-app {
    max-height: 100vh;
    overflow-x: hidden;
  }

  .md-button[disabled] {
    display:none;
  }

  .disabled {
    display:none;
  }

  div.md-app-scroller {
    background-color: #070e1a !important;
    padding: 0px !important;
    margin: 0px !important;
  }

  div.md-app-content {
    background-color: #070e1a !important;
    overflow-x: hidden;
    padding: 0px !important;
    margin: 0px !important;
  }

  img.thumbnail {
    padding: 1px;
    border: 1px solid #070e1a;
    cursor: pointer;
  }

  img.thumbnail:hover {
    border: 1px solid #356dc9;
  }

  .md-empty-state {
    background-color: #356dc9 !important;
  }

  .flex {
    flex: 1;
    display: inline-block;
  }

  .md-app-toolbar {
    -webkit-user-select: none;
    -webkit-app-region: drag;
  }

  button {
    -webkit-app-region: no-drag;
  }

  .md-menu {
    -webkit-app-region: no-drag;
  }

  .md-app-scroller {
    overflow: hidden !important;
  }
</style>

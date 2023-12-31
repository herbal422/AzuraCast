<template>
    <b-modal
        id="apply_playlist_to_modal"
        ref="$modal"
        size="xl"
        centered
        :loading="loading"
        :title="$gettext('Apply Playlist to Folders')"
        @hidden="clearContents"
    >
        <b-overlay
            variant="card"
            :show="loading"
        >
            <div class="form-row">
                <div class="col-md-4">
                    <b-form-fieldset>
                        <b-form-markup id="apply_to_playlist_name">
                            <template #label>
                                {{ $gettext('Playlist:') }}
                            </template>

                            {{ applyToResults.playlist.name }}
                        </b-form-markup>
                    </b-form-fieldset>
                </div>
                <div class="col-md-8">
                    <b-form-fieldset>
                        <b-wrapped-form-checkbox
                            id="form_applyto_copy_playlist"
                            :field="v$.copyPlaylist"
                        >
                            <template #label>
                                {{ $gettext('Create New Playlist for Each Folder') }}
                            </template>
                        </b-wrapped-form-checkbox>
                    </b-form-fieldset>
                </div>
            </div>

            <div style="max-height: 300px; overflow-y: scroll">
                <data-table
                    :fields="fields"
                    :items="applyToResults.directories"
                    :show-toolbar="false"
                    selectable
                    @row-selected="onRowSelected"
                />
            </div>
        </b-overlay>

        <template #modal-footer>
            <b-button
                variant="default"
                @click="close"
            >
                {{ $gettext('Close') }}
            </b-button>
            <b-button
                variant="primary"
                @click="save"
            >
                {{ $gettext('Apply to Folders') }}
            </b-button>
        </template>
    </b-modal>
</template>

<script setup>
import DataTable from '~/components/Common/DataTable.vue';
import {ref} from "vue";
import {useTranslate} from "~/vendor/gettext";
import {useNotify} from "~/vendor/bootstrapVue";
import {useAxios} from "~/vendor/axios";
import BFormFieldset from "~/components/Form/BFormFieldset.vue";
import BFormMarkup from "~/components/Form/BFormMarkup.vue";
import {useVuelidateOnForm} from '~/functions/useVuelidateOnForm';
import {map} from "lodash";
import {useResettableRef} from "~/functions/useResettableRef";
import BWrappedFormCheckbox from "~/components/Form/BWrappedFormCheckbox.vue";

const emit = defineEmits(['relist']);

const $modal = ref(); // Template Ref

const {$gettext} = useTranslate();

let fields = [
    {
        key: 'name',
        isRowHeader: true,
        label: $gettext('Directory')
    }
];

const loading = ref(true);
const applyToUrl = ref(null);
const {record: applyToResults, reset: resetApplyToResults} = useResettableRef({
    playlist: {
        id: null,
        name: ''
    },
    directories: [],
});

const selectedDirs = ref([]);
const onRowSelected = (items) => {
    selectedDirs.value = map(items, 'path');
};

const {form, v$, resetForm} = useVuelidateOnForm(
    {
        copyPlaylist: {}
    },
    {
        copyPlaylist: false
    }
);

const clearContents = () => {
    applyToUrl.value = null;
    selectedDirs.value = [];

    resetApplyToResults();
    resetForm();
};

const {axios} = useAxios();

const open = (newApplyToUrl) => {
    clearContents();

    applyToUrl.value = newApplyToUrl;
    loading.value = true;
    $modal.value?.show();

    axios.get(newApplyToUrl).then((resp) => {
        applyToResults.value = resp.data;
        loading.value = false;
    });
};

const close = () => {
    $modal.value.hide();
};

const {wrapWithLoading, notifySuccess} = useNotify();

const save = () => {
    v$.value.$touch();
    v$.value.$validate().then((isValid) => {
        if (!isValid) {
            return;
        }

        (selectedDirs.value.length) && wrapWithLoading(
            axios.put(applyToUrl.value, {
                ...form.value,
                directories: selectedDirs.value
            })
        ).then(() => {
            notifySuccess($gettext('Playlist successfully applied to folders.'));
        }).finally(() => {
            close();
            emit('relist');
        });
    });
};

defineExpose({
    open
});
</script>

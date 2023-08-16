<template>
  <base-layout>
    <template #header>
      <page-header
        isFilterAvailable
        isSearchAvailable
        showSearchForm
        title="Live Availability"
        v-model:search="liveAvailabilityParams.search"
        @open-filter="openFilterModal"
        :breadcrumbs="breadcrumbs"
      />
    </template>
    <div class="main ion-padding-horizontal">
      <div class="group-button">
        <button
          @click="onlyMyVehicles = false"
          :class="{ active: !onlyMyVehicles }"
          :disabled="loading"
        >
          Vehicles
        </button>
        <button
          @click="onlyMyVehicles = true"
          :class="{ active: !!onlyMyVehicles }"
          :disabled="loading"
        >
          My-Vehicles
        </button>
      </div>
      <div class="google-map" ref="mapRef"></div>
      <div class="legend">
        <span class="legend-item legend-item__available">{{ VehicleStatusesTitlesEnum.AVAILABLE }}</span>
        <span class="legend-item legend-item__part-loaded">{{ VehicleStatusesTitlesEnum.PARTLY_LOADED }}</span>
        <span class="legend-item legend-item__not-available">{{ VehicleStatusesTitlesEnum.NOT_AVAILABLE }}</span>
        <div class="legend-item legend-item__cluster">
          <ion-icon src="assets/icon/cluster-legend.svg"></ion-icon>
          Cluster
        </div>
      </div>
    </div>
    <live-availability-filter-modal
      :is-open="isFilterModalOpen"
      :statuses-filters="statusesFilters"
      :vehicles-filters="vehiclesFilters"
      @close="isFilterModalOpen = false"
      @save="onSaveFilters"
    />
    <tracking-notification-modal
      :is-open="isNotificationModalOpen"
      :is-tracking="isTracking"
      @confirm="onTrackingSubmit"
    />
  </base-layout>
</template>

<script setup lang="ts">
import { onMounted, ref, createApp, reactive, watch, computed } from 'vue';
import { storeToRefs } from 'pinia';
import { useMutation, useQuery } from '@vue/apollo-composable';
import { Loader } from '@googlemaps/js-api-loader';
import { MarkerClusterer } from '@googlemaps/markerclusterer';
import BaseLayout from '@/modules/General/components/BaseLayout.vue';
import PageHeader from '@/modules/General/components/PageHeader.vue';
import TrackingNotificationModal from '../components/TrackingNotificationModal.vue';
import InfoWindow from '../components/InfoWindow.vue';
import { mapOptions } from '@/const/map-options';
import { VehicleStatusesEnum, VehicleStatusesTitlesEnum } from '@/const/vehicleStatuses';
import { ProfileSettingsEnum, Vehicle } from '@/gql/graphql';
import { User } from '@/gql/graphql';
import { Team } from '@/graphql/queries/team';
import { useProfileStore } from '@/modules/Auth/stores/profile';
import { ToggleUserSetting } from '@/graphql/mutations/toggleUserSetting';
import { GetVehicles } from '@/graphql/queries/getVehicles';
import { LiveAvailability } from '@/graphql/queries/liveAvailability';
import LiveAvailabilityFilterModal from '@/modules/LiveAvailability/components/LiveAvailabilityFilterModal.vue';
import { CheckboxFiltersInterface } from '@/modules/General/components/filters/CheckboxFiltersInterface';
import { getCurrentPosition } from '@/helpers/getCurrentPosition';


let google: any;
let map: HTMLElement;
let markerClusterer: any;
const onlyMyVehicles = ref(false);
const profileStore = useProfileStore();
const { profile } = storeToRefs(profileStore);
const mapRef = ref<HTMLDivElement | null>(null);
const isTracking = computed(() => {
  return !!profile.value.profileSettings?.find(setting => setting.setting.code === ProfileSettingsEnum.Tracking)?.value
});

const { result: team } = useQuery(Team, {
  id: profile.value?.team?.id
})

const teamUsers = computed(() => team?.value?.team?.users);

const liveAvailabilityQueryOptions = reactive({
  enabled: false,
  debounce: 500,
  pollInterval: 30000,
  notifyOnNetworkStatusChange: true
});

const vehicleStatuses: Array<VehicleStatusesEnum> = [
  VehicleStatusesEnum.AVAILABLE,
  VehicleStatusesEnum.PARTLY_LOADED,
  VehicleStatusesEnum.NOT_AVAILABLE,
];

const { mutate: toggleTracking } = useMutation(ToggleUserSetting);

const breadcrumbs = {
  items: [
    {
      title: 'Live Availability'
    }
  ]
}

const getPosition = async () => {
  const position = {};
  const coordsOfLondon = { lat: 51.509865, lng: -0.118092 };

  if (isTracking.value) {
    if (navigator.geolocation) {
      const currentPosition = await getCurrentPosition();

      position.lat = liveAvailabilityParams.me_lat = currentPosition.coords.latitude;
      position.lng = liveAvailabilityParams.me_lng = currentPosition.coords.longitude;
    }
  } else {
    position.lat = liveAvailabilityParams.me_lat = coordsOfLondon.lat;
    position.lng = liveAvailabilityParams.me_lng = coordsOfLondon.lng;
  }

  return position;
};

const liveAvailabilityParams = reactive({
  me_lat: '',
  me_lng: '',
  radius: 0,
  search: '',
  availability: vehicleStatuses,
  size: []
});

onMounted(async () => {
  setTimeout(async () => {
    const loader = new Loader({
    apiKey: process.env.VUE_APP_GOOGLE_MAP_API_KEY,
    libraries: ['places', 'marker']
  });

  google = await loader.load();
  await initMap();

  liveAvailabilityParams.radius = getMapRadius();
  liveAvailabilityQueryOptions.enabled = true;
  }, 0);
});

const initMap = async () => {
  map = new google.maps.Map(
      mapRef.value, { ...mapOptions, center: await getPosition() }
  );
}

const getMapRadius = (): number => {
  const mapWidth = mapRef.value.offsetWidth;
  const metersPerPixel = 156543.03392 * Math.cos(map.getCenter().lat() * Math.PI / 180) / Math.pow(2, mapOptions.zoom);
  const mapWidthMeters = mapWidth * metersPerPixel;
  const radius = Math.round(mapWidthMeters / 2 / 1609.344);

  return radius > 40 ? 40 : radius;
}

const { onResult: createVehicleFilters } = useQuery(GetVehicles, { first: 100, page: 1 });

let vehiclesFilters;

createVehicleFilters((vehiclesData) => {
  vehiclesFilters = vehiclesData.data.getVehicles.data
    .filter((vehicle: Vehicle) => vehicle.size !== 1600)
    .map((vehicle: Vehicle) => {
      return {
        value: vehicle.size,
        label: vehicle.name,
        icon: vehicle.icon_url,
        checked: false,
      };
  });
});

const statusesFilters = vehicleStatuses.map((status: VehicleStatusesEnum) => {
  return {
    value: status,
    label: VehicleStatusesTitlesEnum[status],
    checked: false
  };
});

const onSaveFilters = (data: {
  statusesFilters: CheckboxFiltersInterface[];
  vehiclesFilters: CheckboxFiltersInterface[];
}) => {
  liveAvailabilityParams.availability = data.statusesFilters.filter(status => status.checked)
    .map(checkedStatus => checkedStatus.value);

  liveAvailabilityParams.size = data.vehiclesFilters.filter(vehicle => vehicle.checked)
    .map(checkedVehicle => checkedVehicle.value);
};

const markerIconProps = {
  scaledSize: {
    width: 50,
    height: 75,
  },
  labelOrigin: { x: -12, y: -25 },
};

const { onResult: setUpMap, loading, refetch: refetchUsers } = useQuery(
  LiveAvailability,
  { filters: liveAvailabilityParams },
  liveAvailabilityQueryOptions
);

watch(onlyMyVehicles, () => {
  refetchUsers();
});

let currentInfoWindowComponent;
let currentInfoWindow: HTMLDivElement;

const openInfoWindow = (marker, map) => {
  const infoWindowComponent = createApp(
    InfoWindow,
    {
      id: marker.user.id,
      name_first: marker.user.name_first,
      name_last: marker.user.name_last,
      avatar: marker.user.avatar,
      score: marker.user.score,
      vehicleStatus: marker.user.vehicles[0]?.pivot.availability,
    }
  );

  const infoWindowContainer = document.createElement('div');
  infoWindowComponent.mount(infoWindowContainer);
  currentInfoWindowComponent = infoWindowComponent;

  const infoWindow = new google.maps.InfoWindow({
    content: infoWindowContainer,
  });

  infoWindow.open({
    anchor: marker,
    map,
  });

  currentInfoWindow = infoWindow;
};

const filterUsers = (users: User[]) => {
  if (!onlyMyVehicles.value) {
    return users;
  } else {
    return users.filter((user: User) => {
      return !!teamUsers.value.find((relatedUser: User) => user.id === relatedUser.id);
    });
  }
}

setUpMap((liveAvailabilityData) => {
  if (!liveAvailabilityData.data) return;

  interface LiveAvailabilityUser extends User {
    location: { latitude: string, longitude: string  };
  }

  const filteredUsers = filterUsers(liveAvailabilityData.data.liveAvailability);

  const markers = filteredUsers.map((user: LiveAvailabilityUser) => {
    const marker = new google.maps.Marker({
      position: { lat: Number(user.location.latitude), lng: Number(user.location.longitude) },
      icon: {
        ...markerIconProps,
        url: `assets/icon/vehicles-markers/${user.vehicles[0]?.size}_${user.vehicles[0]?.pivot.availability}.svg`},
      user,
      map
    });

    marker.addListener('click', () => {
      currentInfoWindow?.close();
      currentInfoWindowComponent?.unmount();
      openInfoWindow(marker, map);
    });

    return marker;
  });

  if (window.history.state.id) {
    openInfoWindow(markers.find(marker => marker.user.id == window.history.state.id), map)
  }

  google.maps.event.addListener(map, 'click', () => {
    currentInfoWindow?.close();
    currentInfoWindowComponent?.unmount();
  });

  const renderer = {
    render: ({ count, position }) =>
      new google.maps.Marker({
        label: {
          text: String(count),
          color: '#0d5f8f',
          fontSize: '16px',
          fontWeight: '400',
          fontFamily: 'Poppins'
        },
        icon: {
          url: 'assets/icon/cluster-with-circle.svg',
          scaledSize: new google.maps.Size(80, 80),
        },
        position,
        zIndex: Number(google.maps.Marker.MAX_ZINDEX) + count,
      })
  };

  if (!markerClusterer) {
    markerClusterer = new MarkerClusterer({ markers, map, renderer });
  } else {
    markerClusterer.clearMarkers();
    markerClusterer.addMarkers(markers);
  }
});

const isFilterModalOpen = ref<boolean>(false);
const openFilterModal = () => {
  isFilterModalOpen.value = true;
};

const isNotificationModalOpen = ref<boolean>(!isTracking.value && !window.history.state.id);

//check profile store
const onTrackingSubmit = (value: boolean) => {
  if (value !== isTracking.value) {
    toggleTracking({ code: ProfileSettingsEnum.Tracking })
      .then(response => {
        if (!response?.errors) {
          isNotificationModalOpen.value = false;
          profileStore.setData({
            profileSettings: response.data.toggleUserSetting.settings
          });
        }
      });
  } else {
    isNotificationModalOpen.value = false;
  }
}

watch(isTracking, async () => {
  const position = await getPosition();
  const latLng = new google.maps.LatLng(position.lat, position.lng);

  map.panTo(latLng);
});

</script>

<style scoped lang="scss">
.main {
  height: 100%;
  padding: 0;
}

.group-button {
  display: flex;
  position: absolute;
  top: 40px;
  left: 50%;
  transform: translate(-50%, 0);
  z-index: 3;

  button {
    width: 108px;
    height: 31px;
    font-weight: 500;
    font-size: 14px;
    line-height: 1.1;
    letter-spacing: 0.3px;
    background: var(--ion-color-white);
    color: var(--ion-color-black);

    &.active {
      background: var(--ion-color-primary);
      color: var(--ion-color-white);
    }

    &:disabled {
      opacity: 0.5;
    }

    &:first-child {
      border-top-left-radius: 7px;
      border-bottom-left-radius: 7px;
    }

    &:last-child {
      border-top-right-radius: 7px;
      border-bottom-right-radius: 7px;
    }
  }
}

.google-map{
  height: calc(100% + 30px);
  margin-bottom: 25px;
  width: 100%;
}

.legend {
  display: flex;
  justify-content: space-around;
  align-items: center;
  position: absolute;
  width: calc(100% - 22px);
  height: 31px;
  bottom: 30px;
  left: 50%;
  transform: translate(-50%, 0);
  z-index: 3;
  background: var(--ion-color-white);
  border-radius: 7px;

  &-item {
    color: var(--ion-color-primary);
    font-family: var(--ion-font-family);
    font-size: 12px;
    line-height: 18px;

    &:not(.legend__item__cluster):before {
      content: '';
      display: inline-block;
      width: 10px;
      height: 10px;
      margin-right: 5px;
      border-radius: 7px;
    }

    &__available:before {
      background-color: var(--ion-color-success);
    }

    &__part-loaded:before {
      background-color: var(--ion-color-partly-loaded);
    }

    &__not-available:before {
      background-color: var(--ion-color-cancelled);
    }

    &__cluster {
      display: flex;
      align-items: center;

      ion-icon {
        width: 20px;
        height: 20px;
      }
    }
  }
}

:deep(div:has(> img[src='assets/icon/cluster-with-circle.svg'])) {
  overflow: visible !important;

  img {
    top: 21px !important;
  }
}

:deep(div[role='dialog']) {
  left: -50px;

  .gm-style-iw-d {
    overflow: initial;
  }

  button  {
    display: none !important;
  }
}
@media only screen and (min-width: 1023px) {
  .google-map {
    height: calc(100vh - 40px - 27px - 27px - 21px - 25px - 30px - 20px);
  }
}
</style>

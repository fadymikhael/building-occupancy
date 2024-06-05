<template>
  <div id="app" class="min-h-screen bg-gray-100 p-4">
    <header class="bg-gray-800 text-white text-center py-4 mb-6 rounded-lg">
      <h1 class="text-3xl font-bold">Occupation du Bâtiment</h1>
    </header>
    <main class="container mx-auto">
      <!-- Section de filtre pas demandée -->
      <div class="mb-6">
        <label for="filter" class="block text-lg font-medium text-gray-700">Filtrer par état d'occupation:</label>
        <select id="filter" class="mt-2 block w-full py-2 px-3 border border-gray-300 bg-white rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm" v-model="filterStatus">
          <option value="ALL">Tous</option>
          <option value="TRUE">Occupé</option>
          <option value="FALSE">Non Occupé</option>
          <option value="UNDEFINED">Indéfini</option>
        </select>
      </div>
      <!-- Navigation entre les bâtiments -->
      <ul class="nav nav-tabs mb-6" id="myTab" role="tablist">
        <li class="nav-item" v-for="(building, buildingIndex) in buildings" :key="building.name">
          <a :class="['nav-link', { active: buildingIndex === 0 }]" :id="'tab-' + buildingIndex" data-toggle="tab" :href="'#tab-content-' + buildingIndex" role="tab" :aria-controls="'tab-content-' + buildingIndex" :aria-selected="buildingIndex === 0">{{ building.name }}</a>
        </li>
      </ul>
      <!-- Contenu des onglets -->
      <div class="tab-content" id="myTabContent">
        <div v-for="(building, buildingIndex) in buildings" :key="building.name" :class="['tab-pane fade', { 'show active': buildingIndex === 0 }]" :id="'tab-content-' + buildingIndex" role="tabpanel" :aria-labelledby="'tab-' + buildingIndex">
          <!-- Liste des étages et des pièces -->
          <div v-for="(floor, floorIndex) in building.floors" :key="floor.dynamicId" class="mb-6 bg-white rounded-lg shadow-md p-4">
            <div :class="['p-4 rounded-t-lg mb-4 flex justify-between items-center', {'bg-floor text-white': calculateOccupationRate(floor.rooms) !== '0.00', 'bg-light-gray': calculateOccupationRate(floor.rooms) === '0.00'}]">
              <h3 :class="['text-xl font-medium', {'text-white': calculateOccupationRate(floor.rooms) !== '0.00', 'text-gray-800': calculateOccupationRate(floor.rooms) === '0.00'}]">
                Étage Numéro {{ floorIndex + 0 }} : {{ floor.name }}
                <span class="text-sm font-semibold ml-2">
                  (Taux d'occupation: {{ calculateOccupationRate(floor.rooms) }}%)
                </span>
              </h3>
              <span :class="['text-m font-semibold', {'text-white': calculateOccupationRate(floor.rooms) !== '0.00', 'text-gray-800': calculateOccupationRate(floor.rooms) === '0.00'}]">
                Status d'occupation:
              </span>
            </div>
            <ul class="divide-y divide-gray-200 max-h-48 overflow-y-auto pr-2">
              <li v-for="(room, index) in filteredRooms(floor.rooms)" :key="room.dynamicId" class="py-2 flex justify-between items-center">
                <div class="text-lg mr-4">
                  Pièce n°{{ index + 1 }} : {{ room.name }}
                </div>
                <span :class="occupationClass(room.occupation)" class="px-3 py-1 rounded-full text-sm font-medium">
                  {{ room.occupation || 'UNDEFINED' }}
                </span>
              </li>
            </ul>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import { reactive, toRefs } from 'vue'; 
import axios from 'axios'; 

export default {
  setup() {
    const state = reactive({
      buildings: [], 
      filterStatus: 'ALL' 
    });

    // Fonction pour récupérer les données des bâtiments depuis l'API
    const fetchBuildings = async () => {
      try {
        const response = await axios.get('https://api-developers.spinalcom.com/api/v1/geographicContext/space', {
          headers: { accept: 'application/json' }
        });
        const contextData = response.data;
        if (contextData.children && Array.isArray(contextData.children)) {
          const buildings = contextData.children.map(building => ({
            name: building.name,
            floors: building.children ? building.children.map(floor => ({
              name: floor.name,
              dynamicId: String(floor.dynamicId),
              rooms: floor.children ? floor.children.map(room => ({
                name: room.name,
                dynamicId: String(room.dynamicId),
                occupation: 'UNDEFINED' 
              })) : []
            })) : []
          }));
          state.buildings = buildings;
          fetchOccupations(); 
} else {
  console.error('Format de données inattendu pour le contexte:', contextData);
}
} catch (error) {
  console.error('Erreur lors de la récupération des bâtiments:', error);
}

    };

    // Fonction pour récupérer les données d'occupation pour chaque pièce
    const fetchOccupations = async () => {
      const occupationPromises = [];
      state.buildings.forEach(building => {
        building.floors.forEach(floor => {
          floor.rooms.forEach(room => {
            const occupationPromise = axios.get(`https://api-developers.spinalcom.com/api/v1/room/${room.dynamicId}/control_endpoint_list`, {
              headers: { accept: 'application/json' }
            })
            .then(response => {
              if (Array.isArray(response.data)) {
                response.data.forEach(endpointData => {
                  processEndpoints(room, endpointData.endpoints);
                });
              } else if (response.data && typeof response.data === 'object' && response.data.endpoints) {
                processEndpoints(room, response.data.endpoints);
              } else {
                console.error(`Format de données inattendu pour la pièce  ${room.name}:`, response.data);
              }
            })
            .catch(error => {
              console.error(`Erreur lors de la récupération de l'occupation pour la pièce ${room.name}:`, error);
            });
            occupationPromises.push(occupationPromise);
          });
        });
      });
      await Promise.all(occupationPromises); 
    };

    // Fonction pour traiter les données des endpoints et déterminer l'occupation
    const processEndpoints = (room, endpoints) => {
      if (!endpoints || endpoints.length === 0) {
        room.occupation = 'UNDEFINED'; 
        return;
      }
      let found = false;
      endpoints.forEach(endpoint => {
        if (endpoint.type && endpoint.type.toLowerCase() === 'occupation') {
          if (typeof endpoint.currentValue !== 'undefined') {
            room.occupation = endpoint.currentValue ? 'TRUE' : 'FALSE'; // Mise à jour de l'occupation en fonction de la valeur du endpoint
            found = true;
          }
        }
      });
      if (!found) {
        room.occupation = 'UNDEFINED'; 
      }
    };

    // Fonction pour calculer le taux d'occupation d'un étage
    const calculateOccupationRate = (rooms) => {
      const occupiedCount = rooms.filter(room => room.occupation === 'TRUE').length;
      return ((occupiedCount / rooms.length) * 100).toFixed(2); 
    };

    // Fonction pour retourner la classe CSS correspondant à l'occupation
    const occupationClass = (occupation) => {
      return {
        'bg-green-100 text-green-800': occupation === 'TRUE',
        'bg-red-100 text-red-800': occupation === 'FALSE',
        'bg-gray-100 text-gray-800': occupation === 'UNDEFINED'
      };
    };

    // Fonction pour filtrer les pièces en fonction de l'état du filtre
    const filteredRooms = (rooms) => {
      if (state.filterStatus === 'ALL') {
        return rooms;
      }
      return rooms.filter(room => room.occupation === state.filterStatus);
    };

    fetchBuildings(); 

    return {
      ...toRefs(state),
      fetchBuildings,
      fetchOccupations,
      calculateOccupationRate,
      occupationClass,
      filteredRooms
    };
  }
};
</script>

<style>
@import url('https://cdnjs.cloudflare.com/ajax/libs/tailwindcss/2.2.19/tailwind.min.css');
@import url('https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css');

body {
  font-family: 'Inter', sans-serif;
}
.bg-floor {
  background-color: #ffac49;
}

.bg-light-gray {
  background-color: #e0e0e0;
}


ul.max-h-48.overflow-y-auto::-webkit-scrollbar {
  width: 8px;  
}

ul.max-h-48.overflow-y-auto::-webkit-scrollbar-track {
  background: #f1f1f1;  
  border-radius: 10px;
}

ul.max-h-48.overflow-y-auto::-webkit-scrollbar-thumb {
  background: #888;  
  border-radius: 10px;
}

ul.max-h-48.overflow-y-auto::-webkit-scrollbar-thumb:hover {
  background: #555;  
}
</style>

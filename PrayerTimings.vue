<template>
  <div class="flex flex-col items-center justify-center min-h-screen bg-green-500 p-4">
    <!-- Mosque Image -->
    <div class="flex justify-center mb-6">
      <img src="../mosque-clipart-lg.png" alt="Mosque" class="w-1/4 h-auto rounded-lg shadow-md" />
    </div>

    <h1 class="text-2xl text-black font-semibold mb-4 text-center">Prayer Timings</h1>

    <!-- City and Detected Location in a row -->
    <div class="flex justify-center space-x-4 w-full max-w-lg mb-4">
      <!-- City Dropdown -->
      <div class="flex flex-col w-1/2">
        <label for="city" class="block text-4xl font-bold text-center text-black mb-2">Select City</label>
        <select id="city" v-model="city" class="p-2 text-black border border-gray-300 rounded w-full">
          <option value="" disabled>Select city</option>
          <option value="London">London</option>
          <option value="Manchester">Manchester</option>
          <option value="Glasgow">Glasgow</option>
          <option value="Birmingham">Birmingham</option>
          <option value="Liverpool">Liverpool</option>
          <option value="Bradford">Bradford</option>
          <option value="Lahore">Lahore</option>
          <option value="Jhelum">Jhelum</option>
        </select>
      </div>

      <!-- Detected Location Text Box -->
      <div class="flex flex-col w-1/2">
        <label class="block text-4xl font-bold text-center text-black mb-2">Location</label>
        <input
          type="text"
          :value="detectedLocation"
          readonly
          class="p-2 text-black border border-gray-300 rounded w-full"
        />
        <!-- Location Button -->
        <button @click="detectLocation" class="bg-green-700 text-white p-2 mt-1 rounded hover:bg-green-800 w-full">
          Use My Location
        </button>
      </div>
    </div>

    <!-- Date Picker -->
    <div class="flex flex-col mb-4 w-full max-w-xs">
      <label for="date" class="block text-4xl font-bold text-center text-black mb-2">Select Date</label>
      <input type="date" id="date" v-model="date" class="p-2 text-black border border-gray-300 rounded w-full" />
    </div>

    <!-- Get Timings Button -->
    <div class="flex justify-center mb-4">
      <button @click="fetchTimings" class="bg-blue-500 text-white p-2 rounded mt-4 hover:bg-blue-600">
        Get Timings
      </button>
    </div>

    <!-- Error Message -->
    <div v-if="errorMessage" class="text-red-500 mt-4 text-center">
      {{ errorMessage }}
    </div>

    <!-- Prayer Timings Table -->
    <table v-if="timings" class="mt-6 table-auto border-collapse bg-white shadow-md rounded-lg border-4 border-green-600 mx-auto">
      <thead>
        <tr class="bg-blue-100 text-left">
          <th class="px-6 py-3 text-5xl text-center font-semibold text-gray-700 border-b-4 border-green-600">Prayer</th>
          <th class="px-6 py-3 text-5xl text-center font-semibold text-gray-700 border-b-4 border-green-600">Time</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(time, prayer) in timings" :key="prayer" class="hover:bg-gray-100">
          <td class="px-56 py-13 text-4xl text-gray-900 border-b-4 border-green-600">{{ prayer }}</td>
          <td class="px-56 py-13 text-4xl text-gray-900 border-b-4 border-green-600">{{ time }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import { ref } from "vue";
import axios from "axios";

export default {
  name: "PrayerTimings",
  setup() {
    const city = ref("");
    const date = ref("");
    const timings = ref(null);
    const errorMessage = ref("");
    const detectedLocation = ref(""); // New variable for detected location

    const customPrayerNames = {
      Fajr: "Fajr",
      Sunrise: "Sunrise",
      Dhuhr: "Dhuhr",
      Asr: "Asr",
      Maghrib: "Maghrib",
      Isha: "Isha",
      Midnight: "Midnight",
    };

    const detectLocation = () => {
  errorMessage.value = ""; // Clear any previous errors before attempting

  if (!navigator.geolocation) {
    errorMessage.value = "Geolocation is not supported by your browser.";
    return;
  }

  navigator.geolocation.getCurrentPosition(
    async (position) => {
      const { latitude, longitude } = position.coords;
      try {
        const response = await axios.get(
          `https://nominatim.openstreetmap.org/reverse?format=json&lat=${latitude}&lon=${longitude}`
        );
        detectedLocation.value = response.data.address.city || response.data.address.town || response.data.address.village || "Unknown";
        city.value = detectedLocation.value; // Auto select the city in dropdown
      } catch (error) {
        errorMessage.value = "Unable to fetch location.";
      }
    },
    (error) => {
      if (error.code === error.PERMISSION_DENIED) {
        errorMessage.value = "You denied location access. Enable it in browser settings.";
      } else {
        errorMessage.value = "Location access error. Try again later.";
      }
    }
  );
};


    const fetchTimings = async () => {
      if (!city.value || !date.value) {
        errorMessage.value = "Please select both city and date.";
        return;
      }

      try {
        let country = "GB"; // Default to the UK
        if (["Lahore", "Jhelum"].includes(city.value)) {
          country = "PK"; // Pakistan
        }

        const response = await axios.get(`https://api.aladhan.com/v1/timingsByCity`, {
          params: {
            city: city.value,
            country: country,
            method: 2,
            date: date.value.replace(/-/g, "/"),
          },
        });

        if (response.data.code === 200) {
          timings.value = mapPrayerNames(response.data.data.timings);
          errorMessage.value = "";
        } else {
          errorMessage.value = `No timings found for ${city.value} on ${date.value}`;
        }
      } catch (error) {
        errorMessage.value = "An error occurred while fetching data.";
      }
    };

    const mapPrayerNames = (apiTimings) => {
      const mappedTimings = {};
      for (const prayer in apiTimings) {
        if (customPrayerNames[prayer]) {
          mappedTimings[customPrayerNames[prayer]] = apiTimings[prayer];
        }
      }
      return mappedTimings;
    };

    return {
      city,
      date,
      timings,
      errorMessage,
      detectedLocation,
      fetchTimings,
      detectLocation,
    };
  },
};
</script>

<style scoped>
body {
  background-color: white;
}
</style>

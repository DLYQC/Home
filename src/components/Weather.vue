<template>
  <div class="weather" v-if="weatherData.adCode.city && weatherData.weather.weather">
    <span>{{ weatherData.adCode.city }}&nbsp;</span>
    <span>{{ weatherData.weather.weather }}&nbsp;</span>
    <span>{{ weatherData.weather.temperature }}℃</span>
    <span class="sm-hidden">
      &nbsp;{{
        weatherData.weather.winddirection?.endsWith("风")
          ? weatherData.weather.winddirection
          : (weatherData.weather.winddirection || "") + "风"
      }}&nbsp;
    </span>
    <span class="sm-hidden">{{ weatherData.weather.windpower }}&nbsp;级</span>
  </div>
  <div class="weather" v-else>
    <span>天气数据获取失败</span>
  </div>
</template>

<script setup>
import { reactive, onMounted, h } from "vue";
import { getAdcode, getWeather, getOtherWeather } from "@/api";
import { ElMessage } from "element-plus";
import { Error } from "@icon-park/vue-next";

// 高德开发者 Key
const mainKey = import.meta.env.VITE_WEATHER_KEY;

// 天气数据
const weatherData = reactive({
  adCode: {
    city: null, // 城市
    adcode: null, // 城市编码
  },
  weather: {
    weather: null, // 天气现象
    temperature: null, // 实时气温
    winddirection: null, // 风向描述
    windpower: null, // 风力级别
  },
});

// 取出天气平均值（增加类型校验）
const getTemperature = (min, max) => {
  try {
    const numMin = Number(min);
    const numMax = Number(max);
    if (isNaN(numMin) || isNaN(numMax)) {
      throw new Error("温度不是有效数字");
    }
    const average = (numMin + numMax) / 2;
    return Math.round(average);
  } catch (error) {
    console.error("计算温度出现错误：", error);
    return "NaN";
  }
};

// 获取天气数据（增加全链路容错 + 日志输出 + 城市编码兜底）
const getWeatherData = async () => {
  try {
    if (!mainKey) {
      console.log("未配置高德Key，使用备用天气接口");
      const result = await getOtherWeather();
      const data = result?.result || {};
      const cityData = data?.city || {};
      const conditionData = data?.condition || {};

      weatherData.adCode = {
        city: cityData.City || "未知地区",
      };
      weatherData.weather = {
        weather: conditionData.day_weather || "未知",
        temperature: getTemperature(conditionData.min_degree, conditionData.max_degree) || "NaN",
        winddirection: conditionData.day_wind_direction || "未知",
        windpower: conditionData.day_wind_power || "未知",
      };
    } else {
      // 获取 Adcode（增加空值校验 + 城市编码兜底）
      const adCodeData = await getAdcode(mainKey);
      console.log("城市编码原始数据：", adCodeData);
      // 从 city 数组提取第一个城市的编码，若失败则使用武汉编码（430000）兜底
      const adcode = adCodeData?.city?.[0]?.adcode || "430000"; 
      if (typeof adcode !== "string") {
        throw "城市编码格式无效";
      }

      // 获取天气信息（补充 extensions 参数）
      const weatherResult = await getWeather(mainKey, adcode, "base");
      console.log("天气接口返回：", weatherResult);
      if (!weatherResult.lives || weatherResult.lives.length === 0) {
        throw "天气数据为空";
      }
      weatherData.weather = {
        weather: weatherResult.lives[0].weather,
        temperature: weatherResult.lives[0].temperature,
        winddirection: weatherResult.lives[0].winddirection,
        windpower: weatherResult.lives[0].windpower,
      };
      weatherData.adCode.city = adCodeData.city?.[0]?.name || "武汉市";
    }

    // 核心日志：输出最终的 weatherData 状态（判断哪些字段为空）
    console.log("当前天气数据最终状态：", weatherData);
  } catch (error) {
    console.error("天气信息获取失败:", error);
    onError("天气信息获取失败");
  }
};

// 报错信息
const onError = (message) => {
  ElMessage({
    message,
    icon: h(Error, {
      theme: "filled",
      fill: "#efefef",
    }),
  });
  console.error(message);
};

onMounted(() => {
  getWeatherData();
});
</script>
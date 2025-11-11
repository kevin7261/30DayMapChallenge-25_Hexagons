<script>
  /**
   * ═══════════════════════════════════════════════════════════════════════════
   * 🗺️ MapTab.vue - D3.js 台灣地圖組件
   * ═══════════════════════════════════════════════════════════════════════════
   *
   * @fileoverview
   * 這是一個基於 D3.js 的台灣地圖視覺化組件，同時顯示縣市界線和登革熱網格數據。
   * 本組件負責載入、處理和渲染台灣直轄市、縣(市)界線和登革熱病例網格數據。
   *
   * ─────────────────────────────────────────────────────────────────────────
   * 📋 核心功能
   * ─────────────────────────────────────────────────────────────────────────
   * 1. 縣市邊界渲染：
   *    ✓ 載入直轄市、縣(市)界線1140318.geojson
   *    ✓ 繪製所有台灣直轄市、縣(市)界線
   *
   * 2. 登革熱網格渲染：
   *    ✓ 載入 dengue_grid_counts_1km_2023_land_only.geojson
   *    ✓ 根據 level 屬性繪製5級風險等級網格
   *    ✓ 只顯示病例數 > 0 的網格
   *    ✓ 使用5級色票：深藍(1) → 綠(2) → 黃橙(3) → 橙(4) → 紅(5)（最上層）
   *
   * 3. 視覺元素：
   *    ✓ 縣市界線：淺灰細邊框，無填充（底層）
   *    ✓ 登革熱網格：5級色票填充，無邊框（最上層）
   *    ✓ 白色地圖背景
   *
   * 4. 交互功能：
   *    ✓ 滾輪縮放控制
   *    ✓ 拖動平移導航
   *    ✓ 滑鼠懸停顯示網格屬性資訊
   *    ✓ 網格高亮效果
   *
   * ─────────────────────────────────────────────────────────────────────────
   * 🎨 配色主題
   * ─────────────────────────────────────────────────────────────────────────
   * 白色      #ffffff  → 地圖背景
   * 淺灰色    #cccccc  → 縣市邊框
   * 無填充    none     → 縣市區域
   * 5級色票            → 登革熱風險等級（最上層）
   *   Level 1  #1a237e → 深藍色
   *   Level 2  #4caf50 → 綠色
   *   Level 3  #fbc02d → 黃橙色
   *   Level 4  #ff6f00 → 橙色
   *   Level 5  #d32f2f → 紅色
   *
   * ─────────────────────────────────────────────────────────────────────────
   * 🛠️ 技術棧
   * ─────────────────────────────────────────────────────────────────────────
   * @requires vue                 - Vue 3.2+ (Composition API)
   * @requires d3                  - D3.js 7.8+ (地圖繪製庫)
   * @requires @/stores/dataStore  - Pinia 狀態管理
   *
   * ─────────────────────────────────────────────────────────────────────────
   * 📁 數據來源
   * ─────────────────────────────────────────────────────────────────────────
   * 直轄市、縣(市)界線：直轄市、縣(市)界線1140318.geojson
   * 登革熱網格數據：dengue_grid_counts_1km_2023_land_only.geojson
   * 路徑：public/data/geojson/
   *
   * ─────────────────────────────────────────────────────────────────────────
   * 🔧 使用方式
   * ─────────────────────────────────────────────────────────────────────────
   * <MapTab @map-ready="handleMapReady" />
   *
   * @event map-ready - 地圖初始化完成時觸發，返回地圖實例
   *
   * ─────────────────────────────────────────────────────────────────────────
   * 📝 維護者
   * ─────────────────────────────────────────────────────────────────────────
   * @author Kevin Cheng
   * @version 4.0.0
   * @since 2024
   * @license MIT
   *
   * ═══════════════════════════════════════════════════════════════════════════
   */

  // ═══════════════════════════════════════════════════════════════════════════
  // 📦 依賴導入 (Dependencies Import)
  // ═══════════════════════════════════════════════════════════════════════════

  // Vue 3 核心功能
  import { ref, onMounted, onUnmounted, nextTick } from 'vue';

  // D3.js 地圖庫
  import * as d3 from 'd3';

  // Pinia 狀態管理
  import { useDataStore } from '@/stores/dataStore';

  // CesiumJS 3D 地圖庫 - 將使用 CDN 版本，從 window.Cesium 獲取

  // MapLibre GL JS 3D 地圖庫
  import maplibregl from 'maplibre-gl';
  import 'maplibre-gl/dist/maplibre-gl.css';

  // ═══════════════════════════════════════════════════════════════════════════
  // 🎯 組件定義 (Component Definition)
  // ═══════════════════════════════════════════════════════════════════════════

  export default {
    name: 'MapTab',

    // 組件觸發的事件
    emits: [
      'map-ready', // 地圖初始化完成時觸發，傳遞地圖實例
    ],

    /**
     * ───────────────────────────────────────────────────────────────────────
     * 🎬 組件設置函數 (Component Setup Function)
     * ───────────────────────────────────────────────────────────────────────
     * 使用 Vue 3 Composition API 設置組件邏輯
     *
     * @param {Object} _ - Props（本組件不使用）
     * @param {Object} context - 設置上下文
     * @param {Function} context.emit - 事件觸發函數
     * @returns {Object} 返回模板可用的響應式數據和方法
     */
    setup(_, { emit }) {
      // ═══════════════════════════════════════════════════════════════════════
      // 📦 狀態管理與依賴 (State Management & Dependencies)
      // ═══════════════════════════════════════════════════════════════════════

      // Pinia 數據存儲（保留供未來擴展使用）
      // eslint-disable-next-line no-unused-vars
      const dataStore = useDataStore();

      // ═══════════════════════════════════════════════════════════════════════
      // 🗺️ 地圖相關變數 (Map-Related Variables)
      // ═══════════════════════════════════════════════════════════════════════

      /**
       * 地圖 DOM 容器引用
       * @type {Ref<HTMLElement|null>}
       */
      const mapContainer = ref(null);

      /**
       * D3.js SVG 元素
       * @type {d3.Selection|null}
       */
      let svg = null;

      /**
       * D3.js 投影函數
       * @type {d3.GeoProjection|null}
       */
      let projection = null;

      /**
       * D3.js 路徑生成器
       * @type {d3.GeoPath|null}
       */
      let path = null;

      /**
       * D3.js 縮放行為
       * @type {d3.ZoomBehavior|null}
       */
      let zoom = null;

      /**
       * SVG 主容器組
       * @type {d3.Selection|null}
       */
      let g = null;

      /**
       * 工具提示元素
       * @type {HTMLElement|null}
       */
      let tooltip = null;

      /**
       * CesiumJS Viewer 實例
       * @type {any|null}
       */
      let cesiumViewer = null;

      /**
       * MapLibre GL Map 實例
       * @type {maplibregl.Map|null}
       */
      let maplibreMap = null;

      // ═══════════════════════════════════════════════════════════════════════
      // 🎛️ 控制狀態 (Control States)
      // ═══════════════════════════════════════════════════════════════════════

      /**
       * 地圖就緒狀態標記
       * true = 地圖已初始化完成，false = 尚未初始化
       * @type {Ref<boolean>}
       */
      const isMapReady = ref(false);

      /**
       * 地圖容器唯一 ID
       * 使用隨機字符串確保多實例時不會衝突
       * @type {Ref<string>}
       */
      const mapContainerId = ref(`leaflet-map-${Math.random().toString(36).substr(2, 9)}`);

      /**
       * 顯示模式
       * 'map' = 使用地圖投影顯示（目前結果）
       * 'grid' = 直接使用 grid_x, grid_y 繪製網格
       * @type {Ref<string>}
       */
      const displayMode = ref('map');

      // ═══════════════════════════════════════════════════════════════════════
      // 📊 GeoJSON 數據儲存 (GeoJSON Data Storage)
      // ═══════════════════════════════════════════════════════════════════════

      /**
       * 縣市 GeoJSON 數據
       * 來源：直轄市、縣(市)界線1140318.geojson
       * @type {Ref<Object|null>}
       */
      const countyData = ref(null);

      /**
       * 六角形網格 GeoJSON 數據
       * 來源：pointy_final_with_levels_over5.geojson
       * @type {Ref<Object|null>}
       */
      const hexData = ref(null);

      /**
       * 六角形網格 GeoJSON 數據 (模式2)
       * 來源：hex_grid_pointy_with_population.geojson
       * @type {Ref<Object|null>}
       */
      const hexData2 = ref(null);
      const hexData2Stats = ref({
        min: null,
        max: null,
        positiveCount: 0,
      });
      const hexData2Breaks = ref([]);
      const prideColors = [
        '#750787', // Violet
        '#004DFF', // Blue
        '#008026', // Green
        '#FFED00', // Yellow
        '#FF8C00', // Orange
        '#E40303', // Red
      ];
      const createPrideGradientScale = (minVal, maxVal) => {
        const safeMin = Number.isFinite(minVal) ? minVal : 0;
        let safeMax = Number.isFinite(maxVal) ? maxVal : safeMin;
        if (safeMax <= safeMin) {
          safeMax = safeMin === 0 ? 1 : safeMin + Math.abs(safeMin) * 0.01;
        }
        const step = prideColors.length - 1;
        const domain = prideColors.map(
          (_, index) => safeMin + ((safeMax - safeMin) / step) * index
        );
        return d3.scaleLinear().domain(domain).range(prideColors).clamp(true);
      };

      /**
       * 登革熱網格 GeoJSON 數據（保留以兼容）
       * 來源：dengue_grid_counts_1km_2023_land_only.geojson
       * @type {Ref<Object|null>}
       */
      const dengueData = ref(null);

      /**
       * 📥 載入直轄市、縣(市)界線 GeoJSON 數據
       */
      const loadCountyData = async () => {
        try {
          console.log('[MapTab] 開始載入直轄市、縣(市)界線 GeoJSON 數據...');

          // 載入縣市 GeoJSON 檔案
          const countyResponse = await fetch(
            `${process.env.BASE_URL}data/geojson/直轄市、縣(市)界線1140318.geojson`
          );

          // 檢查響應
          if (!countyResponse.ok) {
            throw new Error(`直轄市、縣(市)界線數據載入失敗: HTTP ${countyResponse.status}`);
          }

          // 解析 JSON
          countyData.value = await countyResponse.json();

          console.log('[MapTab] 直轄市、縣(市)界線數據載入成功');
          console.log('  - 縣市數量:', countyData.value.features?.length || 0);

          return true;
        } catch (error) {
          console.error('[MapTab] 直轄市、縣(市)界線數據載入失敗:', error);
          return false;
        }
      };

      /**
       * 🛠️ 創建工具提示元素
       */
      const createTooltip = () => {
        if (!mapContainer.value) return;

        // 移除已存在的工具提示
        const existingTooltip = mapContainer.value.querySelector('.map-tooltip');
        if (existingTooltip) {
          existingTooltip.remove();
        }

        // 創建新的工具提示元素
        tooltip = document.createElement('div');
        tooltip.className = 'map-tooltip';
        tooltip.style.position = 'absolute';
        tooltip.style.pointerEvents = 'none';
        tooltip.style.opacity = '0';
        tooltip.style.padding = '4px 8px';

        mapContainer.value.appendChild(tooltip);
        console.log('[MapTab] 工具提示元素創建成功');
      };

      /**
       * 📥 載入六角形網格 GeoJSON 數據
       */
      const loadHexData = async () => {
        try {
          console.log('[MapTab] 開始載入六角形網格 GeoJSON 數據...');

          // 載入六角形網格 GeoJSON 檔案
          const hexResponse = await fetch(
            `${process.env.BASE_URL}data/geojson/pointy_final_with_levels_over5.geojson`
          );

          // 檢查響應
          if (!hexResponse.ok) {
            throw new Error(`六角形網格數據載入失敗: HTTP ${hexResponse.status}`);
          }

          // 解析 JSON
          hexData.value = await hexResponse.json();

          console.log('[MapTab] 六角形網格數據載入成功');
          console.log('  - 網格數量:', hexData.value.features?.length || 0);

          return true;
        } catch (error) {
          console.error('[MapTab] 六角形網格數據載入失敗:', error);
          return false;
        }
      };

      /**
       * 📥 載入六角形網格 GeoJSON 數據 (模式2)
       */
      const loadHexData2 = async () => {
        try {
          console.log('[MapTab] 開始載入六角形網格 GeoJSON 數據 (模式2)...');

          const hexResponse = await fetch(
            `${process.env.BASE_URL}data/geojson/hex_grid_pointy_with_population.geojson`
          );

          if (!hexResponse.ok) {
            throw new Error(`六角形網格數據 (模式2) 載入失敗: HTTP ${hexResponse.status}`);
          }

          hexData2.value = await hexResponse.json();

          const features = Array.isArray(hexData2.value?.features) ? hexData2.value.features : [];
          const positiveValues = features
            .map((feature) => Number(feature?.properties?.['有偶_相同性別_總計']) || 0)
            .filter((value) => Number.isFinite(value) && value > 0);

          const min = positiveValues.length > 0 ? Math.min(...positiveValues) : 0;
          const max = positiveValues.length > 0 ? Math.max(...positiveValues) : 0;
          hexData2Stats.value = {
            min,
            max,
            positiveCount: positiveValues.length,
          };

          console.log('[MapTab] 六角形網格數據 (模式2) 載入成功');
          console.log('  - 網格數量:', features.length);
          console.log('  - 有效數量（數值 > 0）:', positiveValues.length);
          console.log('  - 數值範圍:', { min, max });

          ensureHexData2Levels();

          return true;
        } catch (error) {
          console.error('[MapTab] 六角形網格數據 (模式2) 載入失敗:', error);
          return false;
        }
      };

      /**
       * 📊 計算自然分級 (Jenks Natural Breaks)
       */
      const calculateNaturalBreaks = (values, classCount) => {
        if (!Array.isArray(values) || values.length === 0) {
          return null;
        }

        const sorted = values.filter((value) => Number.isFinite(value)).sort((a, b) => a - b);

        if (sorted.length === 0) {
          return null;
        }

        const uniqueValues = Array.from(new Set(sorted));
        if (uniqueValues.length <= classCount) {
          const breaks = [uniqueValues[0]];
          for (let i = 1; i < uniqueValues.length; i += 1) {
            breaks.push(uniqueValues[i]);
          }
          while (breaks.length < classCount + 1) {
            breaks.push(uniqueValues[uniqueValues.length - 1]);
          }
          return breaks;
        }

        const lowerClassLimits = Array(sorted.length + 1)
          .fill(0)
          .map(() => Array(classCount + 1).fill(0));
        const varianceCombinations = Array(sorted.length + 1)
          .fill(0)
          .map(() => Array(classCount + 1).fill(0));

        for (let i = 1; i <= classCount; i += 1) {
          lowerClassLimits[1][i] = 1;
          varianceCombinations[1][i] = 0;
          for (let j = 2; j <= sorted.length; j += 1) {
            varianceCombinations[j][i] = Infinity;
          }
        }

        const sum = Array(sorted.length + 1).fill(0);
        const sumSquares = Array(sorted.length + 1).fill(0);

        for (let i = 1; i <= sorted.length; i += 1) {
          const val = sorted[i - 1];
          sum[i] = sum[i - 1] + val;
          sumSquares[i] = sumSquares[i - 1] + val * val;
          varianceCombinations[i][1] = sumSquares[i] - (sum[i] * sum[i]) / i;
          lowerClassLimits[i][1] = 1;
        }

        for (let classes = 2; classes <= classCount; classes += 1) {
          for (let item = classes; item <= sorted.length; item += 1) {
            let variance = Infinity;
            let bestLowerClass = classes - 1;

            for (let i = item; i >= classes; i -= 1) {
              const count = item - i + 1;
              const sumRange = sum[item] - sum[i - 1];
              const sumSquaresRange = sumSquares[item] - sumSquares[i - 1];
              const currentVariance = sumSquaresRange - (sumRange * sumRange) / count;

              if (currentVariance < 0) {
                continue;
              }

              const totalVariance = currentVariance + varianceCombinations[i - 1][classes - 1];

              if (totalVariance < variance) {
                variance = totalVariance;
                bestLowerClass = i;
              }
            }

            varianceCombinations[item][classes] = variance;
            lowerClassLimits[item][classes] = bestLowerClass;
          }
        }

        const breaks = Array(classCount + 1).fill(0);
        breaks[classCount] = sorted[sorted.length - 1];

        let count = classCount;
        let k = sorted.length;

        while (count > 0) {
          const idx = lowerClassLimits[k][count];
          breaks[count - 1] = sorted[idx - 1];
          k = idx - 1;
          count -= 1;
        }

        breaks[0] = sorted[0];

        for (let i = 1; i < breaks.length; i += 1) {
          if (breaks[i] < breaks[i - 1]) {
            breaks[i] = breaks[i - 1];
          }
        }

        return breaks;
      };

      /**
       * 🏷️ 將自然分級結果指派給 features
       */
      const assignLevelsToFeatures = (features, breaks) => {
        if (!Array.isArray(features) || !Array.isArray(breaks) || breaks.length < 2) {
          return;
        }

        features.forEach((feature) => {
          const value = Number(feature?.properties?.['有偶_相同性別_總計']) || 0;
          let level = 0;
          if (value > 0) {
            for (let i = 0; i < breaks.length - 1; i += 1) {
              const lower = breaks[i];
              const upper = breaks[i + 1];
              if (value >= lower && value <= upper) {
                level = i + 1;
                break;
              }
            }
          }
          feature.properties = {
            ...feature.properties,
            pride_level: level,
          };
        });
      };

      /**
       * ♻️ 確保模式2資料具備六色等級
       */
      const ensureHexData2Levels = () => {
        if (!hexData2.value || !Array.isArray(hexData2.value.features)) {
          console.warn('[MapTab] ensureHexData2Levels: hexData2 尚未載入或格式不正確');
          return;
        }

        const features = hexData2.value.features;
        const values = features
          .map((feature) => Number(feature?.properties?.['有偶_相同性別_總計']) || 0)
          .filter((value) => Number.isFinite(value) && value > 0);

        if (values.length === 0) {
          console.warn('[MapTab] ensureHexData2Levels: 無正值可供分級');
          hexData2Breaks.value = [];
          assignLevelsToFeatures(features, []);
          return;
        }

        const classCount = prideColors.length;
        const breaks = calculateNaturalBreaks(values, classCount);

        if (!breaks) {
          console.warn('[MapTab] ensureHexData2Levels: 無法計算自然分級，採用等距分級');
          const min = Math.min(...values);
          const max = Math.max(...values);
          const step = (max - min) / classCount;
          const fallbackBreaks = [min];
          for (let i = 1; i <= classCount; i += 1) {
            fallbackBreaks.push(min + step * i);
          }
          hexData2Breaks.value = fallbackBreaks;
          assignLevelsToFeatures(features, fallbackBreaks);
          return;
        }

        hexData2Breaks.value = breaks;
        assignLevelsToFeatures(features, breaks);
        console.log('[MapTab] ensureHexData2Levels: 計算完成', breaks);
      };

      /**
       * 📥 載入登革熱網格 GeoJSON 數據
       */
      // eslint-disable-next-line no-unused-vars
      const loadDengueData = async () => {
        try {
          console.log('[MapTab] 開始載入登革熱網格 GeoJSON 數據...');

          // 載入登革熱網格 GeoJSON 檔案
          const dengueResponse = await fetch(
            `${process.env.BASE_URL}data/geojson/dengue_grid_counts_1km_2023_land_only.geojson`
          );

          // 檢查響應
          if (!dengueResponse.ok) {
            throw new Error(`登革熱網格數據載入失敗: HTTP ${dengueResponse.status}`);
          }

          // 解析 JSON
          dengueData.value = await dengueResponse.json();

          console.log('[MapTab] 登革熱網格數據載入成功');
          console.log('  - 網格數量:', dengueData.value.features?.length || 0);

          return true;
        } catch (error) {
          console.error('[MapTab] 登革熱網格數據載入失敗:', error);
          return false;
        }
      };

      /**
       * 🗺️ 繪製直轄市、縣(市)界線
       */
      const drawCounties = () => {
        if (!g || !countyData.value) {
          console.error(
            '[MapTab] 無法繪製直轄市、縣(市)界線: g=',
            !!g,
            'countyData=',
            !!countyData.value
          );
          return;
        }

        try {
          console.log('[MapTab] 開始繪製直轄市、縣(市)界線 GeoJSON');

          // 繪製所有縣市
          g.selectAll('.county')
            .data(countyData.value.features)
            .enter()
            .append('path')
            .attr('d', path)
            .attr('class', 'county')
            .attr('fill', 'none') // 不填充
            .attr('stroke', '#cccccc') // 淺灰色邊框
            .attr('stroke-width', 0.5)
            .attr('stroke-opacity', 0.6);

          console.log('[MapTab] 直轄市、縣(市)界線 GeoJSON 繪製完成');
        } catch (error) {
          console.error('[MapTab] 直轄市、縣(市)界線 GeoJSON 繪製失敗:', error);
        }
      };

      /**
       * 🏗️ 創建網格畫布（不依賴地圖投影）
       * 用於 grid 模式，直接使用 grid_x, grid_y 繪製
       */
      // eslint-disable-next-line no-unused-vars
      const createGridCanvas = () => {
        if (!mapContainer.value) return false;

        const rect = mapContainer.value.getBoundingClientRect();
        if (rect.width === 0 || rect.height === 0) {
          console.warn('[MapTab] 容器尺寸為零，延遲初始化');
          return false;
        }

        try {
          // 清除舊的 SVG
          if (svg) {
            // 完全移除 zoom 行為
            try {
              if (zoom) {
                svg.on('.zoom', null);
                svg.call(zoom.on('zoom', null));
              }
            } catch (e) {
              console.warn('[MapTab] 移除 zoom 行為時出錯:', e);
            }
            svg.remove();
            svg = null;
            g = null;
            zoom = null;
          }

          const width = rect.width;
          const height = rect.height;

          // 創建 SVG 元素（不帶地圖投影）
          svg = d3
            .select(mapContainer.value)
            .append('svg')
            .attr('width', width)
            .attr('height', height)
            .style('background', '#ffffff'); // 白色背景

          // 創建容器組（不使用地圖投影）
          g = svg.append('g');

          // 設置縮放行為（用於網格縮放）
          zoom = d3
            .zoom()
            .scaleExtent([0.5, 50]) // 允許縮放 0.5x 到 50x
            .on('zoom', (event) => {
              if (g && g.node() && g.node().parentNode) {
                g.attr('transform', event.transform);
              }
            });

          svg.call(zoom);

          // 重置縮放狀態，確保切換模式時不會受到之前模式的影響
          svg.call(zoom.transform, d3.zoomIdentity);

          // 創建工具提示元素
          createTooltip();

          isMapReady.value = true;

          console.log('[MapTab] 網格畫布創建成功');
          return true;
        } catch (error) {
          console.error('[MapTab] 網格畫布創建失敗:', error);
          return false;
        }
      };

      /**
       * 🗺️ 繪製六角形網格（Grid 模式版本）
       * 使用地圖投影，但沒有縣市界線
       */
      const drawHexGridOnly = () => {
        if (!g || !hexData.value || !path) {
          console.error(
            '[MapTab] 無法繪製六角形網格: g=',
            !!g,
            'hexData=',
            !!hexData.value,
            'path=',
            !!path
          );
          return;
        }

        try {
          console.log('[MapTab] 開始繪製六角形網格（Grid 模式）');

          // 先清除舊的圖層（包括縣市界線）
          g.selectAll('.hex-grid').remove();
          g.selectAll('.county').remove();

          // 過濾掉人口數為0或ratio_China為0或沒有level的區域
          const validFeatures = hexData.value.features.filter(
            (d) =>
              d.properties['人口數'] &&
              d.properties['人口數'] > 0 &&
              d.properties['ratio_China'] &&
              d.properties['ratio_China'] > 0 &&
              d.properties['level'] &&
              d.properties['level'] >= 1 &&
              d.properties['level'] <= 5
          );

          console.log('[MapTab] 使用 level 分類數據:', {
            total: hexData.value.features.length,
            valid: validFeatures.length, // 人口數 > 0 且 ratio_China > 0 且有 level
          });

          // 顏色方案：5級，基於中國國旗紅色 #DE2910 的漸變（淺→深）
          const colors = [
            '#f9d5d3', // level 1 - 很淺（中國紅的淡化版）
            '#f4a9a3', // level 2 - 淺
            '#ee6c5e', // level 3 - 中
            '#de2910', // level 4 - 中國國旗紅
            '#a51f0c', // level 5 - 深（中國紅的深化版）
          ];

          // 顏色映射函數：直接根據 level (1-5) 返回顏色
          const getColor = (level) => {
            if (!level || level < 1 || level > 5) return '#f0f0f0'; // 無數據的顏色
            return colors[level - 1]; // level 1-5 對應 colors[0-4]
          };

          // 計算各級數量（根據 level 統計）
          const classCounts = new Array(colors.length).fill(0);
          validFeatures.forEach((d) => {
            const level = d.properties['level'];
            if (level >= 1 && level <= 5) {
              classCounts[level - 1]++; // level 1-5 對應 classCounts[0-4]
            }
          });

          // 按 level 排序（只考慮人口數 > 0 且 ratio_China > 0 且有 level 的區域）
          const sortedHexes = validFeatures.sort((a, b) => {
            const levelA = a.properties['level'] || 0;
            const levelB = b.properties['level'] || 0;
            return levelA - levelB;
          });

          console.log('[DEBUG] Grid 模式 - 總共要繪製的六角形網格數:', sortedHexes.length);
          console.log(
            '[DEBUG] Grid 模式 - 前 5 個網格:',
            sortedHexes.slice(0, 5).map((d) => ({
              level: d.properties['level'],
              ratio_China: d.properties['ratio_China'],
              color: getColor(d.properties['level']),
            }))
          );

          // 繪製所有六角形網格
          const hexPaths = g
            .selectAll('.hex-grid')
            .data(sortedHexes)
            .enter()
            .append('path')
            .attr('d', path)
            .attr('class', 'hex-grid')
            .attr('fill', (d) => getColor(d.properties['level']))
            .attr('fill-opacity', 0.8)
            .attr('stroke', '#ffffff')
            .attr('stroke-width', 0.5)
            .style('cursor', 'pointer');

          console.log('[DEBUG] Grid 模式 - 繪製了多少個 path 元素:', hexPaths.size());

          hexPaths
            .on('mouseover', function (event, d) {
              d3.select(this).attr('fill-opacity', 1).attr('stroke-width', 2);
              if (tooltip) {
                const properties = d.properties;
                // 格式化小數值為易讀格式
                const formatValue = (key, value) => {
                  if (value === null || value === undefined) return 'N/A';
                  // 對於 ratio 開頭的欄位，格式化為小數
                  if (key.startsWith('ratio_') && typeof value === 'number') {
                    if (value === 0) return '0';
                    if (value < 0.0001) {
                      return value.toExponential(2);
                    }
                    if (value < 0.01) {
                      return value.toFixed(5);
                    }
                    return value.toFixed(4);
                  }
                  return value;
                };
                // 顯示所有 properties 欄位
                let tooltipHTML = '';
                Object.keys(properties).forEach((key) => {
                  const value = properties[key];
                  tooltipHTML += `<div><strong>${key}:</strong> ${formatValue(key, value)}</div>`;
                });
                tooltip.innerHTML = tooltipHTML;
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
                tooltip.style.opacity = 1;
              }
            })
            .on('mousemove', function (event) {
              if (tooltip) {
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
              }
            })
            .on('mouseout', function () {
              d3.select(this).attr('fill-opacity', 0.8).attr('stroke-width', 0.5);
              if (tooltip) {
                tooltip.style.opacity = 0;
              }
            });

          // 繪製圖例
          drawLegend(colors, classCounts);

          console.log('[MapTab] 六角形網格（Grid 模式）繪製完成');
        } catch (error) {
          console.error('[MapTab] 六角形網格繪製失敗:', error);
        }
      };

      /**
       * 🗺️ 繪製網格（使用 grid_x, grid_y，不使用座標）
       * 完全獨立的實現，不依賴地圖投影
       */
      // eslint-disable-next-line no-unused-vars
      const drawGridOnly = () => {
        if (!g || !dengueData.value) {
          console.error('[MapTab] 無法繪製網格: g=', !!g, 'dengueData=', !!dengueData.value);
          return;
        }

        try {
          console.log('[MapTab] 開始繪製網格（使用 grid_x, grid_y）');

          // 清除舊的網格
          g.selectAll('.dengue-grid').remove();

          // 顏色映射
          const levelColors = {
            0: '#e0e0e0', // 淡灰色（level 0）
            1: '#1a237e', // 深藍色（深色）
            2: '#4caf50', // 綠色（較亮）
            3: '#fbc02d', // 黃橙色（金色）
            4: '#ff6f00', // 橙色（明亮）
            5: '#d32f2f', // 紅色（深色）
          };

          // 顏色映射函數
          const getColorByLevel = (level) => {
            if (level === 0 || level === null || level === undefined) {
              return levelColors[0];
            }
            return levelColors[level] || levelColors[1];
          };

          // 透明度映射函數
          const getOpacityByLevel = (level) => {
            const levelNum = level || 0;
            const opacityMap = {
              0: 0.5,
              1: 0.7,
              2: 0.75,
              3: 0.8,
              4: 0.85,
              5: 0.9,
            };
            return opacityMap[levelNum] || opacityMap[0];
          };

          // 過濾有 grid_x 和 grid_y 的數據
          const gridsWithXY = dengueData.value.features.filter(
            (d) =>
              d.properties.grid_x !== null &&
              d.properties.grid_x !== undefined &&
              d.properties.grid_y !== null &&
              d.properties.grid_y !== undefined
          );

          if (gridsWithXY.length === 0) {
            console.error('[MapTab] 無法找到 grid_x 或 grid_y 屬性');
            return;
          }

          // 計算 grid_x 和 grid_y 的範圍
          const gridXValues = gridsWithXY.map((d) => d.properties.grid_x);
          const gridYValues = gridsWithXY.map((d) => d.properties.grid_y);

          const minX = d3.min(gridXValues);
          const maxX = d3.max(gridXValues);
          const minY = d3.min(gridYValues);
          const maxY = d3.max(gridYValues);

          console.log('[MapTab] Grid 範圍:', { minX, maxX, minY, maxY });

          // 獲取 SVG 尺寸
          const svgWidth = +svg.attr('width') || mapContainer.value.getBoundingClientRect().width;
          const svgHeight =
            +svg.attr('height') || mapContainer.value.getBoundingClientRect().height;

          // 創建比例尺（帶有一些邊距）
          const padding = 50;
          const availableWidth = svgWidth - 2 * padding;
          const availableHeight = svgHeight - 2 * padding;

          // 計算 grid 範圍（包括邊界）
          const rangeX = maxX - minX + 1;
          const rangeY = maxY - minY + 1;

          // 計算理論單元大小（根據可用空間和範圍）
          const cellWidthFromX = availableWidth / rangeX;
          const cellHeightFromY = availableHeight / rangeY;

          // 使用較小的值作為統一的單元大小，確保所有網格都是正方形且能完整顯示
          const cellSize = Math.min(cellWidthFromX, cellHeightFromY);

          // 根據實際單元大小計算實際使用的空間
          const actualWidth = cellSize * rangeX;
          const actualHeight = cellSize * rangeY;

          // 計算居中偏移量
          const offsetX = (svgWidth - actualWidth) / 2;
          const offsetY = (svgHeight - actualHeight) / 2;

          // 創建比例尺（使用統一的單元大小，並居中顯示）
          const scaleX = d3
            .scaleLinear()
            .domain([minX, maxX + 1])
            .range([offsetX, offsetX + actualWidth]);
          // Y 軸：grid_y 最小值在上方，最大值在下方（SVG 坐標系：y=0 在頂部，向下遞增）
          const scaleY = d3
            .scaleLinear()
            .domain([minY, maxY + 1])
            .range([offsetY, offsetY + actualHeight]);

          console.log('[MapTab] Grid 單元大小:', {
            cellSize,
            rangeX,
            rangeY,
            cellWidthFromX,
            cellHeightFromY,
          });

          // 網格單元大小（統一為正方形）
          const cellWidth = cellSize;
          const cellHeight = cellSize;

          // 按 level 排序：level 0 在底層，level 1-5 在上層
          const sortedGrids = gridsWithXY.sort((a, b) => {
            const levelA = a.properties.level || 0;
            const levelB = b.properties.level || 0;
            return levelA - levelB;
          });

          // 繪製網格矩形
          g.selectAll('.dengue-grid')
            .data(sortedGrids)
            .enter()
            .append('rect')
            .attr('class', 'dengue-grid')
            .attr('x', (d) => scaleX(d.properties.grid_x))
            .attr('y', (d) => scaleY(d.properties.grid_y))
            .attr('width', cellWidth)
            .attr('height', cellHeight)
            .attr('fill', (d) => getColorByLevel(d.properties.level))
            .attr('fill-opacity', (d) => getOpacityByLevel(d.properties.level))
            .attr('stroke', 'none')
            .style('cursor', 'pointer')
            .on('mouseover', function (event, d) {
              d3.select(this).attr('fill-opacity', 1);
              if (tooltip) {
                const properties = d.properties;
                // 格式化小數值為易讀格式
                const formatValue = (key, value) => {
                  if (value === null || value === undefined) return 'N/A';
                  // 對於 ratio 開頭的欄位，格式化為小數
                  if (key.startsWith('ratio_') && typeof value === 'number') {
                    if (value === 0) return '0';
                    if (value < 0.0001) {
                      return value.toExponential(2);
                    }
                    if (value < 0.01) {
                      return value.toFixed(5);
                    }
                    return value.toFixed(4);
                  }
                  return value;
                };
                // 顯示所有 properties 欄位
                let tooltipHTML = '';
                Object.keys(properties).forEach((key) => {
                  const value = properties[key];
                  tooltipHTML += `<div><strong>${key}:</strong> ${formatValue(key, value)}</div>`;
                });
                tooltip.innerHTML = tooltipHTML;
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
                tooltip.style.opacity = 1;
              }
            })
            .on('mousemove', function (event) {
              if (tooltip) {
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
              }
            })
            .on('mouseout', function (event, d) {
              const level = d.properties.level || 0;
              d3.select(this).attr('fill-opacity', getOpacityByLevel(level));
              if (tooltip) {
                tooltip.style.opacity = 0;
              }
            });

          console.log('[MapTab] 網格繪製完成');
          console.log('  - 網格數量:', sortedGrids.length);
        } catch (error) {
          console.error('[MapTab] 網格繪製失敗:', error);
        }
      };

      /**
       * 🎛️ 切換顯示模式
       * @param {string} mode - 'map' 或 'grid'
       */
      const toggleDisplayMode = async (mode) => {
        displayMode.value = mode;
        console.log('[MapTab] 切換顯示模式:', mode);

        if (displayMode.value === 'map') {
          // 地圖模式：需要地圖投影，載入縣市界線和六角形網格
          if (!countyData.value) {
            await loadCountyData();
          }
          if (!hexData.value) {
            await loadHexData();
          }

          cleanup3DViews();

          // 清除舊的 SVG（如果從其他模式切換過來）
          if (svg && !projection) {
            // 完全移除 zoom 行為
            try {
              if (zoom) {
                svg.on('.zoom', null);
                svg.call(zoom.on('zoom', null));
              }
            } catch (e) {
              console.warn('[MapTab] 移除 zoom 行為時出錯:', e);
            }
            svg.remove();
            svg = null;
            g = null;
            zoom = null;
          }

          if (!projection || !path) {
            // 如果還沒有創建地圖，先創建
            const rect = mapContainer.value.getBoundingClientRect();
            if (rect.width > 0 && rect.height > 0) {
              const width = rect.width;
              const height = rect.height;

              // 清除舊的 SVG
              if (svg) {
                // 取消所有 zoom 事件綁定
                if (zoom) {
                  svg.on('.zoom', null);
                  svg.on('mousedown.zoom', null);
                  svg.on('mousemove.zoom', null);
                  svg.on('mouseup.zoom', null);
                  svg.on('touchstart.zoom', null);
                  svg.on('touchmove.zoom', null);
                  svg.on('touchend.zoom', null);
                  svg.on('wheel.zoom', null);
                }
                svg.remove();
                svg = null;
                g = null;
                zoom = null;
              }

              // 創建 SVG 和地圖投影
              svg = d3
                .select(mapContainer.value)
                .append('svg')
                .attr('width', width)
                .attr('height', height)
                .style('background', '#ffffff');

              projection = d3
                .geoMercator()
                .center([121, 23.5])
                .scale(12000)
                .translate([width / 2, height / 2]);

              path = d3.geoPath().projection(projection);
              g = svg.append('g');

              zoom = d3
                .zoom()
                .scaleExtent([0.5, 50])
                .on('zoom', (event) => {
                  if (g) {
                    g.attr('transform', event.transform);
                  }
                });

              svg.call(zoom);

              // 重置縮放狀態，確保切換模式時不會受到之前模式的影響
              svg.call(zoom.transform, d3.zoomIdentity);

              createTooltip();
              isMapReady.value = true;
            }
          } else {
            // 如果已經創建了地圖，重置縮放狀態
            if (svg && zoom) {
              svg.call(zoom.transform, d3.zoomIdentity);
            }
          }
          // 繪製縣市界線和六角形網格
          drawCounties();
          drawHexGrid();
        } else if (displayMode.value === 'grid') {
          // Grid 模式：載入六角形網格數據，需要地圖投影來繪製
          if (!hexData.value) {
            await loadHexData();
          }
          // 清除縣市界線數據（不需要）
          countyData.value = null;

          cleanup3DViews();

          // 清除舊的 SVG（如果從地圖模式切換過來）
          if (svg && !projection) {
            // 取消 zoom 事件綁定
            if (zoom) {
              svg.on('.zoom', null);
            }
            svg.remove();
            svg = null;
            g = null;
            zoom = null;
          }

          if (!projection || !path) {
            // 如果還沒有創建地圖，先創建
            const rect = mapContainer.value.getBoundingClientRect();
            if (rect.width > 0 && rect.height > 0) {
              const width = rect.width;
              const height = rect.height;

              // 清除舊的 SVG
              if (svg) {
                // 取消所有 zoom 事件綁定
                if (zoom) {
                  svg.on('.zoom', null);
                  svg.on('mousedown.zoom', null);
                  svg.on('mousemove.zoom', null);
                  svg.on('mouseup.zoom', null);
                  svg.on('touchstart.zoom', null);
                  svg.on('touchmove.zoom', null);
                  svg.on('touchend.zoom', null);
                  svg.on('wheel.zoom', null);
                }
                svg.remove();
                svg = null;
                g = null;
                zoom = null;
              }

              // 創建 SVG 和地圖投影（Grid 模式也需要投影來繪製六角形）
              svg = d3
                .select(mapContainer.value)
                .append('svg')
                .attr('width', width)
                .attr('height', height)
                .style('background', '#ffffff');

              projection = d3
                .geoMercator()
                .center([121, 23.5])
                .scale(12000)
                .translate([width / 2, height / 2]);

              path = d3.geoPath().projection(projection);
              g = svg.append('g');

              zoom = d3
                .zoom()
                .scaleExtent([0.5, 50])
                .on('zoom', (event) => {
                  if (g) {
                    g.attr('transform', event.transform);
                  }
                });

              svg.call(zoom);

              // 重置縮放狀態
              svg.call(zoom.transform, d3.zoomIdentity);

              createTooltip();
              isMapReady.value = true;
            }
          } else {
            // 如果已經創建了地圖，重置縮放狀態
            if (svg && zoom) {
              svg.call(zoom.transform, d3.zoomIdentity);
            }
          }

          // 繪製六角形網格（Grid 模式，不顯示縣市界線）
          drawHexGridOnly();
        } else if (displayMode.value === 'cesium3d') {
          // CesiumJS 3D 模式
          if (!hexData.value) {
            await loadHexData();
          }
          if (!countyData.value) {
            await loadCountyData();
          }
          // 清理舊的 SVG 或其他視圖
          cleanupOtherViews();
          // 初始化 CesiumJS 3D 地圖
          await initCesium3D();
        } else if (displayMode.value === 'maplibre3d') {
          // MapLibre 3D 模式
          if (!hexData.value) {
            await loadHexData();
          }
          if (!countyData.value) {
            await loadCountyData();
          }
          // 清理舊的 SVG 或其他視圖
          cleanupOtherViews();
          // 初始化 MapLibre 3D 地圖
          await initMapLibre3D();
        } else if (displayMode.value === 'map2') {
          if (!countyData.value) {
            await loadCountyData();
          }
          if (!hexData2.value) {
            await loadHexData2();
          }

          cleanup3DViews();

          if (svg && !projection) {
            try {
              if (zoom) {
                svg.on('.zoom', null);
                svg.call(zoom.on('zoom', null));
              }
            } catch (e) {
              console.warn('[MapTab] 移除 zoom 行為時出錯:', e);
            }
            svg.remove();
            svg = null;
            g = null;
            zoom = null;
          }

          if (!projection || !path) {
            const rect = mapContainer.value.getBoundingClientRect();
            if (rect.width > 0 && rect.height > 0) {
              const width = rect.width;
              const height = rect.height;

              if (svg) {
                if (zoom) {
                  svg.on('.zoom', null);
                  svg.on('mousedown.zoom', null);
                  svg.on('mousemove.zoom', null);
                  svg.on('mouseup.zoom', null);
                  svg.on('touchstart.zoom', null);
                  svg.on('touchmove.zoom', null);
                  svg.on('touchend.zoom', null);
                  svg.on('wheel.zoom', null);
                }
                svg.remove();
                svg = null;
                g = null;
                zoom = null;
              }

              svg = d3
                .select(mapContainer.value)
                .append('svg')
                .attr('width', width)
                .attr('height', height)
                .style('background', '#000000');

              projection = d3
                .geoMercator()
                .center([121, 23.5])
                .scale(12000)
                .translate([width / 2, height / 2]);

              path = d3.geoPath().projection(projection);
              g = svg.append('g');

              zoom = d3
                .zoom()
                .scaleExtent([0.5, 50])
                .on('zoom', (event) => {
                  if (g) {
                    g.attr('transform', event.transform);
                  }
                });

              svg.call(zoom);
              svg.call(zoom.transform, d3.zoomIdentity);

              createTooltip();
              isMapReady.value = true;
            }
          } else if (svg && zoom) {
            svg.call(zoom.transform, d3.zoomIdentity);
            svg.style('background', '#000000');
          }

          drawCounties();
          drawHexGrid2();
        } else if (displayMode.value === 'grid2') {
          if (!hexData2.value) {
            await loadHexData2();
          }
          countyData.value = null;

          cleanup3DViews();

          if (svg && !projection) {
            if (zoom) {
              svg.on('.zoom', null);
            }
            svg.remove();
            svg = null;
            g = null;
            zoom = null;
          }

          if (!projection || !path) {
            const rect = mapContainer.value.getBoundingClientRect();
            if (rect.width > 0 && rect.height > 0) {
              const width = rect.width;
              const height = rect.height;

              if (svg) {
                if (zoom) {
                  svg.on('.zoom', null);
                  svg.on('mousedown.zoom', null);
                  svg.on('mousemove.zoom', null);
                  svg.on('mouseup.zoom', null);
                  svg.on('touchstart.zoom', null);
                  svg.on('touchmove.zoom', null);
                  svg.on('touchend.zoom', null);
                  svg.on('wheel.zoom', null);
                }
                svg.remove();
                svg = null;
                g = null;
                zoom = null;
              }

              svg = d3
                .select(mapContainer.value)
                .append('svg')
                .attr('width', width)
                .attr('height', height)
                .style('background', '#000000');

              projection = d3
                .geoMercator()
                .center([121, 23.5])
                .scale(12000)
                .translate([width / 2, height / 2]);

              path = d3.geoPath().projection(projection);
              g = svg.append('g');

              zoom = d3
                .zoom()
                .scaleExtent([0.5, 50])
                .on('zoom', (event) => {
                  if (g) {
                    g.attr('transform', event.transform);
                  }
                });

              svg.call(zoom);
              svg.call(zoom.transform, d3.zoomIdentity);

              createTooltip();
              isMapReady.value = true;
            }
          } else if (svg && zoom) {
            svg.call(zoom.transform, d3.zoomIdentity);
            svg.style('background', '#000000');
          }

          drawHexGridOnly2();
        } else if (displayMode.value === 'cesium3d2') {
          if (!hexData2.value) {
            await loadHexData2();
          } else {
            ensureHexData2Levels();
          }
          if (!countyData.value) {
            await loadCountyData();
          }
          cleanupOtherViews();
          await initCesium3D2();
        } else if (displayMode.value === 'maplibre3d2') {
          if (!hexData2.value) {
            await loadHexData2();
          } else {
            ensureHexData2Levels();
          }
          if (!countyData.value) {
            await loadCountyData();
          }
          cleanupOtherViews();
          await initMapLibre3D2();
        }
      };

      /**
       * 🧹 只清理 3D 視圖（Cesium / MapLibre），保留 D3 狀態
       */
      const cleanup3DViews = () => {
        let cleared = false;

        if (cesiumViewer) {
          try {
            cesiumViewer.destroy();
          } catch (e) {
            console.warn('[MapTab] 清理 Cesium Viewer 時出錯:', e);
          }
          cesiumViewer = null;
          cleared = true;
        }

        if (maplibreMap) {
          try {
            maplibreMap.remove();
          } catch (e) {
            console.warn('[MapTab] 清理 MapLibre Map 時出錯:', e);
          }
          maplibreMap = null;
          cleared = true;
        }

        if (cleared && mapContainer.value) {
          mapContainer.value.innerHTML = '';
          svg = null;
          g = null;
          zoom = null;
          projection = null;
          path = null;
          if (tooltip) {
            tooltip.remove();
            tooltip = null;
          }
        }
      };

      /**
       * 🧹 清理其他視圖（SVG、Cesium、MapLibre）
       */
      const cleanupOtherViews = () => {
        // 清理 SVG
        if (svg) {
          try {
            if (zoom) {
              svg.on('.zoom', null);
            }
            svg.remove();
          } catch (e) {
            console.warn('[MapTab] 清理 SVG 時出錯:', e);
          }
          svg = null;
          g = null;
          zoom = null;
          projection = null;
          path = null;
        }

        // 清理 Cesium Viewer
        if (cesiumViewer) {
          try {
            cesiumViewer.destroy();
          } catch (e) {
            console.warn('[MapTab] 清理 Cesium Viewer 時出錯:', e);
          }
          cesiumViewer = null;
        }

        // 清理 MapLibre Map
        if (maplibreMap) {
          try {
            maplibreMap.remove();
          } catch (e) {
            console.warn('[MapTab] 清理 MapLibre Map 時出錯:', e);
          }
          maplibreMap = null;
        }

        // 清空容器
        if (mapContainer.value) {
          mapContainer.value.innerHTML = '';
        }
      };

      /**
       * 🌍 初始化 CesiumJS 3D 地圖
       */
      const initCesium3D = async () => {
        try {
          console.log('[MapTab] 開始初始化 CesiumJS 3D 地圖');

          if (!mapContainer.value || !hexData.value) {
            console.error('[MapTab] 無法初始化 CesiumJS: 容器或數據不存在');
            return;
          }

          // 載入縣市界線數據（如果尚未載入）
          if (!countyData.value) {
            await loadCountyData();
          }

          // 檢查 Cesium 是否已載入（從 CDN）
          if (typeof window.Cesium === 'undefined') {
            console.error('[MapTab] CesiumJS 尚未載入，請確保已引入 CDN 腳本');
            return;
          }

          // eslint-disable-next-line no-undef
          const Cesium = window.Cesium;

          // 設置 Cesium Ion 訪問令牌
          Cesium.Ion.defaultAccessToken =
            'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiIxYjJiZjlhZC1mZDNkLTRiZWEtYjExNy1iZDI1OWQ5ZmJlZmEiLCJpZCI6MzU1MDgxLCJpYXQiOjE3NjE3MTc5NTl9.ivNUz20WJNOvyTB6vzB8xHNWNSzgl06vBAGOuZLNKs4';

          // 創建世界地形提供者（需要 Ion token）
          const worldTerrain = await Cesium.createWorldTerrainAsync();

          // 創建 Cesium Viewer（使用世界地形）
          cesiumViewer = new Cesium.Viewer(mapContainer.value, {
            terrainProvider: worldTerrain,
            baseLayerPicker: false,
            geocoder: false,
            homeButton: false,
            infoBox: true,
            sceneModePicker: false,
            selectionIndicator: false,
            timeline: false,
            animation: false,
            fullscreenButton: false,
            vrButton: false,
            navigationHelpButton: false,
          });

          // 設置視角到台灣
          cesiumViewer.camera.setView({
            destination: Cesium.Cartesian3.fromDegrees(121.0, 23.5, 500000),
            orientation: {
              heading: 0.0,
              pitch: -0.5,
              roll: 0.0,
            },
          });

          // 顏色方案：根據 level (1-5) 返回顏色
          const getColorByLevel = (level) => {
            const colors = {
              1: Cesium.Color.fromCssColorString('#f9d5d3'), // level 1 - 很淺
              2: Cesium.Color.fromCssColorString('#f4a9a3'), // level 2 - 淺
              3: Cesium.Color.fromCssColorString('#ee6c5e'), // level 3 - 中
              4: Cesium.Color.fromCssColorString('#de2910'), // level 4 - 中國國旗紅
              5: Cesium.Color.fromCssColorString('#a51f0c'), // level 5 - 深
            };
            return colors[level] || Cesium.Color.fromCssColorString('#f0f0f0');
          };

          // 計算六角形網格的寬度（米）
          // 從第一個有效的六角形計算寬度（相對頂點之間的距離）
          const calculateHexWidth = () => {
            if (!hexData.value || !hexData.value.features || hexData.value.features.length === 0) {
              return 6000; // 默認值，如果無法計算
            }

            const firstHex = hexData.value.features.find(
              (f) =>
                f.properties.level >= 1 &&
                f.properties.level <= 5 &&
                f.geometry &&
                f.geometry.coordinates[0]
            );

            if (!firstHex || !firstHex.geometry.coordinates[0]) {
              return 6000; // 默認值
            }

            const coords = firstHex.geometry.coordinates[0];
            if (coords.length < 4) {
              return 6000; // 默認值
            }

            // 計算相對頂點之間的距離（六角形寬度）
            const p1 = coords[0];
            const p4 = coords[3]; // 相對頂點

            // 使用 Haversine 公式計算距離（米）
            const R = 6371000; // 地球半徑（米）
            const lat1 = (p1[1] * Math.PI) / 180;
            const lat4 = (p4[1] * Math.PI) / 180;
            const dLat = ((p4[1] - p1[1]) * Math.PI) / 180;
            const dLon = ((p4[0] - p1[0]) * Math.PI) / 180;
            const a =
              Math.sin(dLat / 2) * Math.sin(dLat / 2) +
              Math.cos(lat1) * Math.cos(lat4) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
            const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
            const width = R * c;

            console.log('[MapTab] CesiumJS 3D - 計算的六角形寬度（米）:', width);
            return width;
          };

          const hexWidth = calculateHexWidth();

          // 將 level 轉換為高度（米）
          // level 1 的高度 = 六角形寬度
          // level 5 的高度 = level 1 的 5 倍 = 六角形寬度的 5 倍
          const levelToHeight = (level) => {
            if (!level || level < 1 || level > 5) return 0;
            // level 1 = hexWidth, level 2 = 2 * hexWidth, ..., level 5 = 5 * hexWidth
            return level * hexWidth;
          };

          // 過濾有效的 features（level 1-5）
          const validFeatures = hexData.value.features.filter(
            (d) =>
              d.properties.level &&
              d.properties.level >= 1 &&
              d.properties.level <= 5 &&
              d.geometry &&
              d.geometry.type === 'Polygon'
          );

          console.log('[MapTab] CesiumJS 3D - 有效的 features 數量:', validFeatures.length);

          // 繪製台灣縣市界線（白色邊框）
          if (countyData.value && countyData.value.features) {
            countyData.value.features.forEach((countyFeature) => {
              if (countyFeature.geometry && countyFeature.geometry.type === 'Polygon') {
                const coordinates = countyFeature.geometry.coordinates[0];
                const positions = coordinates.map((coord) =>
                  Cesium.Cartesian3.fromDegrees(coord[0], coord[1], 0)
                );

                // 創建縣市界線（白色邊框，無填充）
                cesiumViewer.entities.add({
                  polyline: {
                    positions: positions.concat([positions[0]]), // 閉合多邊形
                    width: 2.0,
                    material: Cesium.Color.WHITE.withAlpha(0.8),
                    clampToGround: true, // 貼地顯示
                  },
                });
              } else if (countyFeature.geometry && countyFeature.geometry.type === 'MultiPolygon') {
                // 處理 MultiPolygon 類型
                countyFeature.geometry.coordinates.forEach((polygon) => {
                  const coordinates = polygon[0];
                  const positions = coordinates.map((coord) =>
                    Cesium.Cartesian3.fromDegrees(coord[0], coord[1], 0)
                  );

                  cesiumViewer.entities.add({
                    polyline: {
                      positions: positions.concat([positions[0]]),
                      width: 2.0,
                      material: Cesium.Color.WHITE.withAlpha(0.8),
                      clampToGround: true,
                    },
                  });
                });
              }
            });
            console.log('[MapTab] CesiumJS 3D - 縣市界線繪製完成');
          }

          // 為每個 feature 創建 3D 柱狀圖（高度由 level 決定）
          validFeatures.forEach((feature) => {
            const level = feature.properties.level;
            const coordinates = feature.geometry.coordinates[0]; // Polygon 的第一個環
            const extrudedHeight = levelToHeight(level);

            // 將 GeoJSON 座標轉換為 Cesium 的 Cartesian3 數組
            // 基礎座標應該在地面（高度 0），然後通過 extrudedHeight 向上擠壓
            const positions = coordinates.map((coord) =>
              Cesium.Cartesian3.fromDegrees(coord[0], coord[1], 0)
            );

            // 創建擠壓多邊形實體（柱狀圖效果）
            cesiumViewer.entities.add({
              polygon: {
                hierarchy: positions,
                material: getColorByLevel(level).withAlpha(0.8),
                // 擠壓高度：從地面（height=0）到目標高度（extrudedHeight）
                height: 0, // 基礎高度（地面）
                extrudedHeight: extrudedHeight, // 擠壓到的高度
                // 不顯示邊框
                outline: false,
              },
              properties: feature.properties,
            });
          });

          // 格式化 village_list 為 HTML
          const formatVillageList = (villageList) => {
            if (!villageList) return 'N/A';

            // 如果是字符串，嘗試解析
            let villages = villageList;
            if (typeof villageList === 'string') {
              try {
                // 先嘗試標準 JSON 解析
                villages = JSON.parse(villageList);
              } catch (e) {
                try {
                  // 如果是 Python 格式的字符串（使用單引號），先轉換為標準 JSON
                  // 將單引號改為雙引號，None 改為 null
                  const jsonString = villageList
                    .replace(/'/g, '"')
                    .replace(/None/g, 'null')
                    .replace(/True/g, 'true')
                    .replace(/False/g, 'false');
                  villages = JSON.parse(jsonString);
                } catch (e2) {
                  return villageList; // 如果解析失敗，直接返回字符串
                }
              }
            }

            // 如果不是數組，返回原值
            if (!Array.isArray(villages)) {
              return String(villages);
            }

            // 格式化每個 village 項目
            if (villages.length === 0) return '無資料';

            return villages
              .map((village, index) => {
                if (typeof village === 'object' && village !== null) {
                  const items = Object.keys(village)
                    .map((key) => {
                      const val = village[key];
                      const displayVal = val === null || val === undefined ? 'N/A' : val;
                      return `${key}: ${displayVal}`;
                    })
                    .join(', ');
                  return `${index + 1}. ${items}`;
                }
                return `${index + 1}. ${village}`;
              })
              .join('<br>');
          };

          // 添加點擊事件顯示屬性信息
          const handler = new Cesium.ScreenSpaceEventHandler(cesiumViewer.scene.canvas);
          handler.setInputAction((click) => {
            const pickedObject = cesiumViewer.scene.pick(click.position);
            if (Cesium.defined(pickedObject) && pickedObject.id && pickedObject.id.properties) {
              const properties = pickedObject.id.properties;
              let html = '<div style="max-width: 400px; max-height: 500px; overflow-y: auto;">';

              // 顯示所有屬性
              for (const key in properties) {
                if (Object.prototype.hasOwnProperty.call(properties, key)) {
                  const value = properties[key].getValue();

                  if (key === 'village_list' || key === 'villageList') {
                    // 特殊處理 village_list
                    html += `<div><strong>${key}:</strong><br>${formatVillageList(value)}</div><br>`;
                  } else {
                    html += `<div><strong>${key}:</strong> ${value}</div>`;
                  }
                }
              }

              html += '</div>';

              // 使用 Cesium InfoBox 顯示
              console.log('[MapTab] CesiumJS - 點擊的實體信息:', html);

              // 顯示在 infoBox 中
              if (cesiumViewer.infoBox && pickedObject.id) {
                cesiumViewer.selectedEntity = pickedObject.id;
                cesiumViewer.infoBox.viewModel.description = html;
              }
            }
          }, Cesium.ScreenSpaceEventType.LEFT_CLICK);

          isMapReady.value = true;
          console.log('[MapTab] CesiumJS 3D 地圖初始化完成');
        } catch (error) {
          console.error('[MapTab] CesiumJS 3D 地圖初始化失敗:', error);
        }
      };

      /**
       * 🗺️ 初始化 MapLibre GL 3D 地圖
       */
      const initMapLibre3D = async () => {
        try {
          console.log('[MapTab] 開始初始化 MapLibre GL 3D 地圖');

          if (!mapContainer.value || !hexData.value) {
            console.error('[MapTab] 無法初始化 MapLibre GL: 容器或數據不存在');
            return;
          }

          // 載入縣市界線數據（如果尚未載入）
          if (!countyData.value) {
            await loadCountyData();
          }

          // 創建 MapLibre Map（使用 Carto Dark 底圖）
          maplibreMap = new maplibregl.Map({
            container: mapContainer.value,
            style: {
              version: 8,
              sources: {
                carto: {
                  type: 'raster',
                  tiles: [
                    'https://a.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png',
                    'https://b.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png',
                    'https://c.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png',
                  ],
                  tileSize: 256,
                  attribution:
                    '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
                },
              },
              layers: [
                {
                  id: 'carto-dark-layer',
                  type: 'raster',
                  source: 'carto',
                  minzoom: 0,
                  maxzoom: 22,
                },
              ],
            },
            center: [121.0, 23.5], // 台灣中心
            zoom: 8,
            pitch: 45, // 傾斜角度，創建 3D 效果
            bearing: 0,
            antialias: true,
          });

          // 等待地圖載入完成
          maplibreMap.on('load', () => {
            console.log('[MapTab] MapLibre GL 地圖載入完成');

            // 計算六角形網格的寬度（米）
            // 從第一個有效的六角形計算寬度（相對頂點之間的距離）
            const calculateHexWidth = () => {
              if (
                !hexData.value ||
                !hexData.value.features ||
                hexData.value.features.length === 0
              ) {
                return 6000; // 默認值，如果無法計算
              }

              const firstHex = hexData.value.features.find(
                (f) =>
                  f.properties.level >= 1 &&
                  f.properties.level <= 5 &&
                  f.geometry &&
                  f.geometry.coordinates[0]
              );

              if (!firstHex || !firstHex.geometry.coordinates[0]) {
                return 6000; // 默認值
              }

              const coords = firstHex.geometry.coordinates[0];
              if (coords.length < 4) {
                return 6000; // 默認值
              }

              // 計算相對頂點之間的距離（六角形寬度）
              const p1 = coords[0];
              const p4 = coords[3]; // 相對頂點

              // 使用 Haversine 公式計算距離（米）
              const R = 6371000; // 地球半徑（米）
              const lat1 = (p1[1] * Math.PI) / 180;
              const lat4 = (p4[1] * Math.PI) / 180;
              const dLat = ((p4[1] - p1[1]) * Math.PI) / 180;
              const dLon = ((p4[0] - p1[0]) * Math.PI) / 180;
              const a =
                Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                Math.cos(lat1) * Math.cos(lat4) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
              const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
              const width = R * c;

              console.log('[MapTab] MapLibre GL 3D - 計算的六角形寬度（米）:', width);
              return width;
            };

            const hexWidth = calculateHexWidth();

            // 將 level 轉換為高度（米）
            // level 1 的高度 = 六角形寬度
            // level 5 的高度 = level 1 的 5 倍 = 六角形寬度的 5 倍
            const levelToHeight = (level) => {
              if (!level || level < 1 || level > 5) return 0;
              // level 1 = hexWidth, level 2 = 2 * hexWidth, ..., level 5 = 5 * hexWidth
              return level * hexWidth;
            };

            // 顏色方案：根據 level (1-5) 返回顏色
            const getColorByLevel = (level) => {
              const colors = {
                1: '#f9d5d3', // level 1 - 很淺
                2: '#f4a9a3', // level 2 - 淺
                3: '#ee6c5e', // level 3 - 中
                4: '#de2910', // level 4 - 中國國旗紅
                5: '#a51f0c', // level 5 - 深
              };
              return colors[level] || '#f0f0f0';
            };

            // 準備 GeoJSON 數據，為每個 feature 添加高度屬性
            const featuresWithHeight = hexData.value.features
              .filter(
                (d) =>
                  d.properties.level &&
                  d.properties.level >= 1 &&
                  d.properties.level <= 5 &&
                  d.geometry &&
                  d.geometry.type === 'Polygon'
              )
              .map((feature) => {
                const level = feature.properties.level;
                const height = levelToHeight(level);

                // 保持原始座標格式 [lng, lat]，高度通過屬性設置
                return {
                  ...feature,
                  properties: {
                    ...feature.properties,
                    base_height: height,
                    color: getColorByLevel(level),
                  },
                };
              });

            console.log(
              '[MapTab] MapLibre GL 3D - 有效的 features 數量:',
              featuresWithHeight.length
            );

            // 添加縣市界線 GeoJSON 源
            if (countyData.value && countyData.value.features) {
              maplibreMap.addSource('county-boundaries', {
                type: 'geojson',
                data: countyData.value,
              });

              // 添加縣市界線圖層（白色邊框）
              maplibreMap.addLayer({
                id: 'county-boundaries-line',
                type: 'line',
                source: 'county-boundaries',
                paint: {
                  'line-color': '#ffffff',
                  'line-width': 2,
                  'line-opacity': 0.8,
                },
              });
              console.log('[MapTab] MapLibre GL 3D - 縣市界線繪製完成');
            }

            // 添加 GeoJSON 源
            maplibreMap.addSource('hexagons-3d', {
              type: 'geojson',
              data: {
                type: 'FeatureCollection',
                features: featuresWithHeight,
              },
            });

            // 添加填充圖層
            maplibreMap.addLayer({
              id: 'hexagons-3d-fill',
              type: 'fill-extrusion',
              source: 'hexagons-3d',
              paint: {
                'fill-extrusion-color': ['get', 'color'],
                'fill-extrusion-height': ['get', 'base_height'],
                'fill-extrusion-base': 0,
                'fill-extrusion-opacity': 0.8,
              },
            });

            // 不添加邊框圖層（移除白色邊框）

            // 格式化 village_list 為 HTML
            const formatVillageList = (villageList) => {
              if (!villageList) return 'N/A';

              // 如果是字符串，嘗試解析
              let villages = villageList;
              if (typeof villageList === 'string') {
                try {
                  // 先嘗試標準 JSON 解析
                  villages = JSON.parse(villageList);
                } catch (e) {
                  try {
                    // 如果是 Python 格式的字符串（使用單引號），先轉換為標準 JSON
                    // 將單引號改為雙引號，None 改為 null
                    const jsonString = villageList
                      .replace(/'/g, '"')
                      .replace(/None/g, 'null')
                      .replace(/True/g, 'true')
                      .replace(/False/g, 'false');
                    villages = JSON.parse(jsonString);
                  } catch (e2) {
                    return villageList; // 如果解析失敗，直接返回字符串
                  }
                }
              }

              // 如果不是數組，返回原值
              if (!Array.isArray(villages)) {
                return String(villages);
              }

              // 格式化每個 village 項目
              if (villages.length === 0) return '無資料';

              return villages
                .map((village, index) => {
                  if (typeof village === 'object' && village !== null) {
                    const items = Object.keys(village)
                      .map((key) => {
                        const val = village[key];
                        const displayVal = val === null || val === undefined ? 'N/A' : val;
                        return `<span style="margin-right: 10px;"><strong>${key}:</strong> ${displayVal}</span>`;
                      })
                      .join('');
                    return `<div style="margin: 5px 0; padding: 5px; background: rgba(255,255,255,0.1); border-radius: 3px;">${index + 1}. ${items}</div>`;
                  }
                  return `<div style="margin: 5px 0;">${index + 1}. ${village}</div>`;
                })
                .join('');
            };

            // 添加點擊事件
            maplibreMap.on('click', 'hexagons-3d-fill', (e) => {
              const properties = e.features[0].properties;
              console.log('[MapTab] MapLibre GL - 點擊的實體信息:', properties);

              // 格式化所有屬性為 HTML
              let html = '<div style="max-width: 400px; max-height: 500px; overflow-y: auto;">';

              Object.keys(properties).forEach((key) => {
                const value = properties[key];

                if (key === 'village_list' || key === 'villageList') {
                  // 特殊處理 village_list
                  html += `<div style="margin-bottom: 10px;"><strong>${key}:</strong><div style="margin-top: 5px;">${formatVillageList(value)}</div></div>`;
                } else {
                  html += `<div style="margin-bottom: 5px;"><strong>${key}:</strong> ${value}</div>`;
                }
              });

              html += '</div>';

              // 創建彈出框
              new maplibregl.Popup({ maxWidth: '450px' })
                .setLngLat(e.lngLat)
                .setHTML(html)
                .addTo(maplibreMap);
            });

            // 改變鼠標樣式
            maplibreMap.on('mouseenter', 'hexagons-3d-fill', () => {
              maplibreMap.getCanvas().style.cursor = 'pointer';
            });

            maplibreMap.on('mouseleave', 'hexagons-3d-fill', () => {
              maplibreMap.getCanvas().style.cursor = '';
            });

            isMapReady.value = true;
            console.log('[MapTab] MapLibre GL 3D 地圖初始化完成');
          });

          maplibreMap.on('error', (error) => {
            console.error('[MapTab] MapLibre GL 地圖載入錯誤:', error);
          });
        } catch (error) {
          console.error('[MapTab] MapLibre GL 3D 地圖初始化失敗:', error);
        }
      };

      /**
       * 🌍 初始化 CesiumJS 3D 地圖 (模式2)
       */
      const initCesium3D2 = async () => {
        try {
          console.log('[MapTab] 開始初始化 CesiumJS 3D 地圖（模式2）');

          if (!mapContainer.value || !hexData2.value) {
            console.error('[MapTab] 無法初始化 CesiumJS（模式2）: 容器或數據不存在');
            return;
          }

          if (!countyData.value) {
            await loadCountyData();
          }

          if (typeof window.Cesium === 'undefined') {
            console.error('[MapTab] CesiumJS 尚未載入，請確保已引入 CDN 腳本');
            return;
          }

          // eslint-disable-next-line no-undef
          const Cesium = window.Cesium;

          Cesium.Ion.defaultAccessToken =
            'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiIxYjJiZjlhZC1mZDNkLTRiZWEtYjExNy1iZDI1OWQ5ZmJlZmEiLCJpZCI6MzU1MDgxLCJpYXQiOjE3NjE3MTc5NTl9.ivNUz20WJNOvyTB6vzB8xHNWNSzgl06vBAGOuZLNKs4';

          const worldTerrain = await Cesium.createWorldTerrainAsync();

          cesiumViewer = new Cesium.Viewer(mapContainer.value, {
            terrainProvider: worldTerrain,
            baseLayerPicker: false,
            geocoder: false,
            homeButton: false,
            infoBox: true,
            sceneModePicker: false,
            selectionIndicator: false,
            timeline: false,
            animation: false,
            fullscreenButton: false,
            vrButton: false,
            navigationHelpButton: false,
          });

          cesiumViewer.camera.setView({
            destination: Cesium.Cartesian3.fromDegrees(121.0, 23.5, 500000),
            orientation: {
              heading: 0.0,
              pitch: -0.5,
              roll: 0.0,
            },
          });

          const calculateHexWidth = () => {
            if (
              !hexData2.value ||
              !hexData2.value.features ||
              hexData2.value.features.length === 0
            ) {
              return 6000;
            }

            const firstHex = hexData2.value.features.find(
              (f) =>
                f.geometry && f.geometry.coordinates?.[0] && f.geometry.coordinates[0].length >= 4
            );

            if (!firstHex || !firstHex.geometry.coordinates[0]) {
              return 6000;
            }

            const coords = firstHex.geometry.coordinates[0];
            const p1 = coords[0];
            const p4 = coords[3];

            const R = 6371000;
            const lat1 = (p1[1] * Math.PI) / 180;
            const lat4 = (p4[1] * Math.PI) / 180;
            const dLat = ((p4[1] - p1[1]) * Math.PI) / 180;
            const dLon = ((p4[0] - p1[0]) * Math.PI) / 180;
            const a =
              Math.sin(dLat / 2) * Math.sin(dLat / 2) +
              Math.cos(lat1) * Math.cos(lat4) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
            const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
            const width = R * c;

            console.log('[MapTab] CesiumJS 3D（模式2）- 計算的六角形寬度（米）:', width);
            return width;
          };

          const hexWidth = calculateHexWidth();

          const valueKey = '有偶_相同性別_總計';
          const getValue = (feature) => Number(feature?.properties?.[valueKey]) || 0;

          const { min, max } = hexData2Stats.value || {};
          const domainMin = Number.isFinite(min) && min > 0 ? min : 0;
          const rawMax = Number.isFinite(max) && max > 0 ? max : domainMin;
          const effectiveMax = rawMax > domainMin ? rawMax : domainMin === 0 ? 1 : domainMin * 1.01;

          const heightScale = d3
            .scaleLinear()
            .domain([domainMin, effectiveMax])
            .range([hexWidth * 0.5, hexWidth * 8]);

          const colorScale = createPrideGradientScale(domainMin, effectiveMax);

          const features = Array.isArray(hexData2.value.features) ? hexData2.value.features : [];
          const validFeatures = features.filter(
            (feature) =>
              feature.geometry && feature.geometry.type === 'Polygon' && getValue(feature) > 0
          );

          if (countyData.value && countyData.value.features) {
            countyData.value.features.forEach((countyFeature) => {
              if (countyFeature.geometry && countyFeature.geometry.type === 'Polygon') {
                const coordinates = countyFeature.geometry.coordinates[0];
                const positions = coordinates.map((coord) =>
                  Cesium.Cartesian3.fromDegrees(coord[0], coord[1], 0)
                );

                cesiumViewer.entities.add({
                  polyline: {
                    positions: positions.concat([positions[0]]),
                    width: 2.0,
                    material: Cesium.Color.WHITE.withAlpha(0.8),
                    clampToGround: true,
                  },
                });
              } else if (countyFeature.geometry && countyFeature.geometry.type === 'MultiPolygon') {
                countyFeature.geometry.coordinates.forEach((polygon) => {
                  const coordinates = polygon[0];
                  const positions = coordinates.map((coord) =>
                    Cesium.Cartesian3.fromDegrees(coord[0], coord[1], 0)
                  );

                  cesiumViewer.entities.add({
                    polyline: {
                      positions: positions.concat([positions[0]]),
                      width: 2.0,
                      material: Cesium.Color.WHITE.withAlpha(0.8),
                      clampToGround: true,
                    },
                  });
                });
              }
            });
          }

          validFeatures.forEach((feature) => {
            const value = getValue(feature);
            const color = Cesium.Color.fromCssColorString(
              colorScale(Math.min(effectiveMax, Math.max(domainMin, value)))
            ).withAlpha(1);
            const coordinates = feature.geometry.coordinates[0];
            const extrudedHeight = heightScale(Math.min(effectiveMax, Math.max(domainMin, value)));
            const hierarchy = coordinates.map((coord) =>
              Cesium.Cartesian3.fromDegrees(coord[0], coord[1], 0)
            );

            cesiumViewer.entities.add({
              polygon: {
                hierarchy,
                material: color,
                height: 0,
                extrudedHeight,
                outline: false,
              },
              properties: feature.properties,
            });
          });

          const formatVillageList = (villageList) => {
            if (!villageList) return 'N/A';
            let villages = villageList;
            if (typeof villageList === 'string') {
              try {
                villages = JSON.parse(villageList);
              } catch (e) {
                try {
                  const jsonString = villageList
                    .replace(/'/g, '"')
                    .replace(/None/g, 'null')
                    .replace(/True/g, 'true')
                    .replace(/False/g, 'false');
                  villages = JSON.parse(jsonString);
                } catch (e2) {
                  return villageList;
                }
              }
            }

            if (!Array.isArray(villages)) {
              return String(villages);
            }

            if (villages.length === 0) return '無資料';

            return villages
              .map((village, index) => {
                if (typeof village === 'object' && village !== null) {
                  const items = Object.keys(village)
                    .map((key) => {
                      const val = village[key];
                      const displayVal = val === null || val === undefined ? 'N/A' : val;
                      return `${key}: ${displayVal}`;
                    })
                    .join(', ');
                  return `${index + 1}. ${items}`;
                }
                return `${index + 1}. ${village}`;
              })
              .join('<br>');
          };

          const handler = new Cesium.ScreenSpaceEventHandler(cesiumViewer.scene.canvas);
          handler.setInputAction((click) => {
            const pickedObject = cesiumViewer.scene.pick(click.position);
            if (Cesium.defined(pickedObject) && pickedObject.id && pickedObject.id.properties) {
              const properties = pickedObject.id.properties;
              let html = '<div style="max-width: 420px; max-height: 520px; overflow-y: auto;">';

              for (const key in properties) {
                if (Object.prototype.hasOwnProperty.call(properties, key)) {
                  const propertyValue = properties[key];
                  const value =
                    propertyValue &&
                    typeof propertyValue === 'object' &&
                    typeof propertyValue.getValue === 'function'
                      ? propertyValue.getValue()
                      : propertyValue;

                  if (key === 'village_list' || key === 'villageList') {
                    html += `<div><strong>${key}:</strong><br>${formatVillageList(value)}</div><br>`;
                  } else {
                    html += `<div><strong>${key}:</strong> ${value}</div>`;
                  }
                }
              }

              html += '</div>';

              if (cesiumViewer.infoBox && pickedObject.id) {
                cesiumViewer.selectedEntity = pickedObject.id;
                cesiumViewer.infoBox.viewModel.description = html;
              }
            }
          }, Cesium.ScreenSpaceEventType.LEFT_CLICK);

          isMapReady.value = true;
          console.log('[MapTab] CesiumJS 3D 地圖（模式2）初始化完成');
        } catch (error) {
          console.error('[MapTab] CesiumJS 3D 地圖（模式2）初始化失敗:', error);
        }
      };

      /**
       * 🗺️ 初始化 MapLibre GL 3D 地圖 (模式2)
       */
      const initMapLibre3D2 = async () => {
        try {
          console.log('[MapTab] 開始初始化 MapLibre GL 3D 地圖（模式2）');

          if (!mapContainer.value || !hexData2.value) {
            console.error('[MapTab] 無法初始化 MapLibre GL（模式2）: 容器或數據不存在');
            return;
          }

          if (!countyData.value) {
            await loadCountyData();
          }

          maplibreMap = new maplibregl.Map({
            container: mapContainer.value,
            style: {
              version: 8,
              sources: {
                carto: {
                  type: 'raster',
                  tiles: [
                    'https://a.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png',
                    'https://b.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png',
                    'https://c.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png',
                  ],
                  tileSize: 256,
                  attribution:
                    '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
                },
              },
              layers: [
                {
                  id: 'carto-dark-layer',
                  type: 'raster',
                  source: 'carto',
                  minzoom: 0,
                  maxzoom: 22,
                },
              ],
            },
            center: [121.0, 23.5],
            zoom: 8,
            pitch: 45,
            bearing: 0,
            antialias: true,
          });

          maplibreMap.on('load', () => {
            console.log('[MapTab] MapLibre GL 地圖（模式2）載入完成');

            const calculateHexWidth = () => {
              if (
                !hexData2.value ||
                !hexData2.value.features ||
                hexData2.value.features.length === 0
              ) {
                return 6000;
              }

              const firstHex = hexData2.value.features.find(
                (f) =>
                  f.geometry && f.geometry.coordinates?.[0] && f.geometry.coordinates[0].length >= 4
              );

              if (!firstHex || !firstHex.geometry.coordinates[0]) {
                return 6000;
              }

              const coords = firstHex.geometry.coordinates[0];
              const p1 = coords[0];
              const p4 = coords[3];

              const R = 6371000;
              const lat1 = (p1[1] * Math.PI) / 180;
              const lat4 = (p4[1] * Math.PI) / 180;
              const dLat = ((p4[1] - p1[1]) * Math.PI) / 180;
              const dLon = ((p4[0] - p1[0]) * Math.PI) / 180;
              const a =
                Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                Math.cos(lat1) * Math.cos(lat4) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
              const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
              const width = R * c;

              console.log('[MapTab] MapLibre GL 3D（模式2）- 計算的六角形寬度（米）:', width);
              return width;
            };

            const hexWidth = calculateHexWidth();

            const valueKey = '有偶_相同性別_總計';
            const getValue = (feature) => Number(feature?.properties?.[valueKey]) || 0;

            const { min, max } = hexData2Stats.value || {};
            const domainMin = Number.isFinite(min) && min > 0 ? min : 0;
            const rawMax = Number.isFinite(max) && max > 0 ? max : domainMin;
            const effectiveMax =
              rawMax > domainMin ? rawMax : domainMin === 0 ? 1 : domainMin * 1.01;

            const heightScale = d3
              .scaleLinear()
              .domain([domainMin, effectiveMax])
              .range([hexWidth * 0.5, hexWidth * 8]);

            const colorScale = createPrideGradientScale(domainMin, effectiveMax);

            const features = Array.isArray(hexData2.value.features) ? hexData2.value.features : [];
            const featuresWithHeight = features
              .filter(
                (feature) =>
                  feature.geometry && feature.geometry.type === 'Polygon' && getValue(feature) > 0
              )
              .map((feature) => {
                const value = getValue(feature);
                return {
                  ...feature,
                  properties: {
                    ...feature.properties,
                    base_height: heightScale(Math.min(effectiveMax, Math.max(domainMin, value))),
                    color: colorScale(Math.min(effectiveMax, Math.max(domainMin, value))),
                  },
                };
              });

            if (countyData.value && countyData.value.features) {
              maplibreMap.addSource('county-boundaries', {
                type: 'geojson',
                data: countyData.value,
              });

              maplibreMap.addLayer({
                id: 'county-boundaries-line',
                type: 'line',
                source: 'county-boundaries',
                paint: {
                  'line-color': '#ffffff',
                  'line-width': 2,
                  'line-opacity': 0.8,
                },
              });
              console.log('[MapTab] MapLibre GL 3D（模式2）- 縣市界線繪製完成');
            }

            maplibreMap.addSource('hexagons-3d', {
              type: 'geojson',
              data: {
                type: 'FeatureCollection',
                features: featuresWithHeight,
              },
            });

            maplibreMap.addLayer({
              id: 'hexagons-3d-fill',
              type: 'fill-extrusion',
              source: 'hexagons-3d',
              paint: {
                'fill-extrusion-color': ['get', 'color'],
                'fill-extrusion-height': ['get', 'base_height'],
                'fill-extrusion-base': 0,
                'fill-extrusion-opacity': 1,
              },
            });

            const formatVillageList = (villageList) => {
              if (!villageList) return 'N/A';
              let villages = villageList;
              if (typeof villageList === 'string') {
                try {
                  villages = JSON.parse(villageList);
                } catch (e) {
                  try {
                    const jsonString = villageList
                      .replace(/'/g, '"')
                      .replace(/None/g, 'null')
                      .replace(/True/g, 'true')
                      .replace(/False/g, 'false');
                    villages = JSON.parse(jsonString);
                  } catch (e2) {
                    return villageList;
                  }
                }
              }

              if (!Array.isArray(villages)) {
                return String(villages);
              }

              if (villages.length === 0) return '無資料';

              return villages
                .map((village, index) => {
                  if (typeof village === 'object' && village !== null) {
                    const items = Object.keys(village)
                      .map((key) => {
                        const val = village[key];
                        const displayVal = val === null || val === undefined ? 'N/A' : val;
                        return `<span style="margin-right: 10px;"><strong>${key}:</strong> ${displayVal}</span>`;
                      })
                      .join('');
                    return `<div style="margin: 5px 0; padding: 5px; background: rgba(255,255,255,0.1); border-radius: 3px;">${index + 1}. ${items}</div>`;
                  }
                  return `<div style="margin: 5px 0;">${index + 1}. ${village}</div>`;
                })
                .join('');
            };

            maplibreMap.on('click', 'hexagons-3d-fill', (e) => {
              const properties = e.features[0].properties;
              console.log('[MapTab] MapLibre GL（模式2）- 點擊的實體信息:', properties);

              let html = '<div style="max-width: 420px; max-height: 520px; overflow-y: auto;">';

              Object.keys(properties).forEach((key) => {
                const value = properties[key];

                if (key === 'village_list' || key === 'villageList') {
                  html += `<div style="margin-bottom: 10px;"><strong>${key}:</strong><div style="margin-top: 5px;">${formatVillageList(value)}</div></div>`;
                } else {
                  html += `<div style="margin-bottom: 5px;"><strong>${key}:</strong> ${value}</div>`;
                }
              });

              html += '</div>';

              new maplibregl.Popup({ maxWidth: '450px' })
                .setLngLat(e.lngLat)
                .setHTML(html)
                .addTo(maplibreMap);
            });

            maplibreMap.on('mouseenter', 'hexagons-3d-fill', () => {
              maplibreMap.getCanvas().style.cursor = 'pointer';
            });

            maplibreMap.on('mouseleave', 'hexagons-3d-fill', () => {
              maplibreMap.getCanvas().style.cursor = '';
            });

            isMapReady.value = true;
            console.log('[MapTab] MapLibre GL 3D 地圖（模式2）初始化完成');
          });

          maplibreMap.on('error', (error) => {
            console.error('[MapTab] MapLibre GL 地圖（模式2）載入錯誤:', error);
          });
        } catch (error) {
          console.error('[MapTab] MapLibre GL 3D 地圖（模式2）初始化失敗:', error);
        }
      };

      /**
       * 🗺️ 繪製六角形網格（使用 ratio_China 數據）
       */
      const drawHexGrid = () => {
        if (!g || !hexData.value || !path) {
          console.error(
            '[MapTab] 無法繪製六角形網格: g=',
            !!g,
            'hexData=',
            !!hexData.value,
            'path=',
            !!path
          );
          return;
        }

        try {
          console.log('[MapTab] 開始繪製六角形網格 GeoJSON');

          // 先清除舊的圖層
          g.selectAll('.hex-grid').remove();

          // 過濾掉人口數為0或ratio_China為0或沒有level的區域
          const validFeatures = hexData.value.features.filter(
            (d) =>
              d.properties['人口數'] &&
              d.properties['人口數'] > 0 &&
              d.properties['ratio_China'] &&
              d.properties['ratio_China'] > 0 &&
              d.properties['level'] &&
              d.properties['level'] >= 1 &&
              d.properties['level'] <= 5
          );

          console.log('[MapTab] 使用 level 分類數據:', {
            total: hexData.value.features.length,
            valid: validFeatures.length, // 人口數 > 0 且 ratio_China > 0 且有 level
          });

          // 顏色方案：5級，基於中國國旗紅色 #DE2910 的漸變（淺→深）
          const colors = [
            '#f9d5d3', // level 1 - 很淺（中國紅的淡化版）
            '#f4a9a3', // level 2 - 淺
            '#ee6c5e', // level 3 - 中
            '#de2910', // level 4 - 中國國旗紅
            '#a51f0c', // level 5 - 深（中國紅的深化版）
          ];

          // 顏色映射函數：直接根據 level (1-5) 返回顏色
          const getColor = (level) => {
            if (!level || level < 1 || level > 5) return '#f0f0f0'; // 無數據的顏色
            return colors[level - 1]; // level 1-5 對應 colors[0-4]
          };

          // 計算各級數量（根據 level 統計）
          const classCounts = new Array(colors.length).fill(0);
          validFeatures.forEach((d) => {
            const level = d.properties['level'];
            if (level >= 1 && level <= 5) {
              classCounts[level - 1]++; // level 1-5 對應 classCounts[0-4]
            }
          });

          // 按 level 排序（只考慮人口數 > 0 且 ratio_China > 0 且有 level 的區域）
          const sortedHexes = validFeatures.sort((a, b) => {
            const levelA = a.properties['level'] || 0;
            const levelB = b.properties['level'] || 0;
            return levelA - levelB;
          });

          console.log('[DEBUG] 總共要繪製的六角形網格數:', sortedHexes.length);
          console.log(
            '[DEBUG] 前 5 個網格:',
            sortedHexes.slice(0, 5).map((d) => ({
              level: d.properties['level'],
              ratio_China: d.properties['ratio_China'],
              color: getColor(d.properties['level']),
            }))
          );

          // Map 模式：使用地圖投影繪製（使用 GeoJSON coordinates）
          console.log('[MapTab] 使用 Map 模式繪製（地圖投影）');
          console.log('[MapTab] path generator:', !!path, 'g:', !!g);

          // 繪製所有六角形網格
          const hexPaths = g
            .selectAll('.hex-grid')
            .data(sortedHexes)
            .enter()
            .append('path')
            .attr('d', path)
            .attr('class', 'hex-grid')
            .attr('fill', (d) => getColor(d.properties['level']))
            .attr('fill-opacity', 0.8)
            .attr('stroke', '#ffffff')
            .attr('stroke-width', 0.5)
            .style('cursor', 'pointer');

          console.log('[DEBUG] 繪製了多少個 path 元素:', hexPaths.size());

          hexPaths
            .on('mouseover', function (event, d) {
              d3.select(this).attr('fill-opacity', 1).attr('stroke-width', 2);
              if (tooltip) {
                const properties = d.properties;
                // 格式化小數值為易讀格式
                const formatValue = (key, value) => {
                  if (value === null || value === undefined) return 'N/A';
                  // 對於 ratio 開頭的欄位，格式化為小數
                  if (key.startsWith('ratio_') && typeof value === 'number') {
                    if (value === 0) return '0';
                    if (value < 0.0001) {
                      return value.toExponential(2);
                    }
                    if (value < 0.01) {
                      return value.toFixed(5);
                    }
                    return value.toFixed(4);
                  }
                  return value;
                };
                // 顯示所有 properties 欄位
                let tooltipHTML = '';
                Object.keys(properties).forEach((key) => {
                  const value = properties[key];
                  tooltipHTML += `<div><strong>${key}:</strong> ${formatValue(key, value)}</div>`;
                });
                tooltip.innerHTML = tooltipHTML;
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
                tooltip.style.opacity = 1;
              }
            })
            .on('mousemove', function (event) {
              if (tooltip) {
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
              }
            })
            .on('mouseout', function () {
              d3.select(this).attr('fill-opacity', 0.8).attr('stroke-width', 0.5);
              if (tooltip) {
                tooltip.style.opacity = 0;
              }
            });

          // 繪製圖例
          drawLegend(colors, classCounts);

          console.log('[MapTab] 六角形網格（地圖模式）繪製完成');
          console.log('  - 使用 level 分類 (1-5)');
          console.log('  - SVG 中的 path 元素數量:', g.selectAll('path').size());
          console.log('  - hex-grid class 元素數量:', g.selectAll('.hex-grid').size());
        } catch (error) {
          console.error('[MapTab] 登革熱網格繪製失敗:', error);
        }
      };

      /**
       * 🎨 繪製圖例
       */
      const drawLegend = (colors, classCounts) => {
        if (!svg || !mapContainer.value) return;

        // 移除舊的圖例
        svg.selectAll('.legend').remove();

        // 圖例尺寸（增加寬度和高度以改善間距）
        const legendWidth = 280;
        const legendHeight = 24;
        const padding = 15;
        const labelSpacing = 28; // 標籤之間的垂直間距

        // 計算右下角位置（使用容器實際尺寸）
        const rect = mapContainer.value.getBoundingClientRect();
        const svgWidth = rect.width;
        const svgHeight = rect.height;
        const legendX = svgWidth - legendWidth - padding;
        const legendY = svgHeight - legendHeight - 120; // 留出更多標籤空間

        // 創建圖例組（固定在 viewport，不受 zoom 影響）
        const legend = svg
          .append('g')
          .attr('class', 'legend')
          .attr('data-legend-x', legendX)
          .attr('data-legend-y', legendY)
          .attr('transform', `translate(${legendX}, ${legendY})`);

        // 繪製每個顏色塊
        legend
          .selectAll('.legend-color')
          .data(colors)
          .enter()
          .append('rect')
          .attr('class', 'legend-color')
          .attr('x', (d, i) => i * (legendWidth / colors.length))
          .attr('y', 0)
          .attr('width', legendWidth / colors.length)
          .attr('height', legendHeight)
          .attr('fill', (d) => d)
          .attr('stroke', '#333')
          .attr('stroke-width', 1);

        // 添加 level 標籤 (1-5)
        const levels = [1, 2, 3, 4, 5];
        legend
          .selectAll('.legend-label')
          .data(levels)
          .enter()
          .append('text')
          .attr('class', 'legend-label')
          .attr('x', (d, i) => (i + 0.5) * (legendWidth / levels.length))
          .attr('y', legendHeight + labelSpacing)
          .attr('font-size', '12px')
          .attr('fill', '#333')
          .attr('text-anchor', 'middle')
          .text((d) => `Level ${d}`);

        // 添加各級數量標籤
        if (classCounts) {
          legend
            .selectAll('.legend-count')
            .data(classCounts)
            .enter()
            .append('text')
            .attr('class', 'legend-count')
            .attr('x', (d, i) => (i + 0.5) * (legendWidth / classCounts.length))
            .attr('y', legendHeight + labelSpacing * 2)
            .attr('font-size', '11px')
            .attr('fill', '#666')
            .attr('text-anchor', 'middle')
            .text((d) => d);
        }

        // 添加標題
        legend
          .append('text')
          .attr('class', 'legend-title')
          .attr('x', legendWidth / 2)
          .attr('y', -12)
          .attr('font-size', '13px')
          .attr('font-weight', 'bold')
          .attr('fill', '#333')
          .attr('text-anchor', 'middle')
          .text('ratio_China (比例值) - Level');
      };

      /**
       * 🌈 Pride 六色圖例
       */
      const drawPrideGradientLegend = (min, max, colorScale) => {
        if (!svg || !mapContainer.value) return;

        svg.selectAll('.legend').remove();

        const rect = mapContainer.value.getBoundingClientRect();
        const svgWidth = rect.width;
        const svgHeight = rect.height;
        const legendWidth = 260;
        const boxHeight = 16;
        const legendX = svgWidth - legendWidth - 15;
        const legendY = svgHeight - boxHeight - 120;

        const legend = svg
          .append('g')
          .attr('class', 'legend')
          .attr('transform', `translate(${legendX}, ${legendY})`);

        legend
          .append('text')
          .attr('x', legendWidth / 2)
          .attr('y', -10)
          .attr('text-anchor', 'middle')
          .attr('font-size', '13px')
          .attr('font-weight', 'bold')
          .attr('fill', '#f5f5f5')
          .text('有偶_相同性別_總計');

        let start = Number.isFinite(min) ? min : 0;
        let end = Number.isFinite(max) ? max : start;
        if (end <= start) {
          end = start === 0 ? 1 : start + Math.abs(start) * 0.01;
        }

        let defs = svg.select('defs');
        if (defs.empty()) {
          defs = svg.append('defs');
        }

        defs.select('#pride-gradient').remove();
        const gradient = defs
          .append('linearGradient')
          .attr('id', 'pride-gradient')
          .attr('x1', '0%')
          .attr('y1', '0%')
          .attr('x2', '100%')
          .attr('y2', '0%');

        prideColors.forEach((_, index) => {
          const t = index / (prideColors.length - 1);
          const value = start + t * (end - start);
          gradient
            .append('stop')
            .attr('offset', `${t * 100}%`)
            .attr('stop-color', colorScale(value));
        });

        legend
          .append('rect')
          .attr('width', legendWidth)
          .attr('height', boxHeight)
          .attr('fill', 'url(#pride-gradient)')
          .attr('stroke', '#f5f5f5')
          .attr('stroke-width', 1)
          .attr('rx', 4)
          .attr('ry', 4);

        const scale = d3.scaleLinear().domain([start, end]).range([0, legendWidth]);
        const axis = d3
          .axisBottom(scale)
          .ticks(6)
          .tickFormat((d) => Math.round(d).toLocaleString());

        legend
          .append('g')
          .attr('transform', `translate(0, ${boxHeight})`)
          .call(axis)
          .selectAll('text')
          .style('font-size', '11px')
          .style('fill', '#f5f5f5');

        legend.selectAll('.domain, .tick line').style('stroke', '#f5f5f5');
      };

      /**
       * 🗺️ Pride 六色地圖（模式2）
       */
      const drawHexGrid2 = () => {
        if (!g || !hexData2.value || !path) {
          console.error(
            '[MapTab] 無法繪製六角形網格（模式2）: g=',
            !!g,
            'hexData2=',
            !!hexData2.value,
            'path=',
            !!path
          );
          return;
        }

        try {
          console.log('[MapTab] 開始繪製六角形網格（模式2）GeoJSON');

          g.selectAll('.hex-grid').remove();

          const features = Array.isArray(hexData2.value.features) ? hexData2.value.features : [];
          const valueKey = '有偶_相同性別_總計';
          const getValue = (feature) => Number(feature?.properties?.[valueKey]) || 0;

          const { min, max } = hexData2Stats.value || {};
          const domainMin = Number.isFinite(min) && min > 0 ? min : 0;
          const domainMax = Number.isFinite(max) && max > 0 ? max : domainMin > 0 ? domainMin : 1;
          const colorScale = createPrideGradientScale(domainMin, domainMax);

          const validFeatures = features.filter((feature) => getValue(feature) > 0);

          if (validFeatures.length === 0) {
            console.warn('[MapTab] 模式2 無可用的 Pride 六角格');
            g.selectAll('.legend').remove();
            return;
          }

          const sortedHexes = [...validFeatures].sort((a, b) => getValue(a) - getValue(b));

          console.log('[DEBUG] 模式2 - 有效六角格數量:', sortedHexes.length);
          console.log('[DEBUG] 模式2 - 數值範圍:', { min: domainMin, max: domainMax });

          const hexPaths = g
            .selectAll('.hex-grid')
            .data(sortedHexes)
            .enter()
            .append('path')
            .attr('d', path)
            .attr('class', 'hex-grid')
            .attr('fill', (d) => colorScale(Math.min(domainMax, Math.max(domainMin, getValue(d)))))
            .attr('fill-opacity', 0.85)
            .attr('stroke', '#ffffff')
            .attr('stroke-width', 0.5)
            .style('cursor', 'pointer');

          hexPaths
            .on('mouseover', function (event, d) {
              d3.select(this).attr('fill-opacity', 1).attr('stroke-width', 2);
              if (tooltip) {
                const properties = d.properties || {};
                const formatValue = (key, value) => {
                  if (value === null || value === undefined) return 'N/A';
                  if (typeof value === 'number') {
                    return value.toLocaleString();
                  }
                  return value;
                };
                let tooltipHTML = '';
                Object.keys(properties).forEach((key) => {
                  const value = properties[key];
                  tooltipHTML += `<div><strong>${key}:</strong> ${formatValue(key, value)}</div>`;
                });
                tooltip.innerHTML = tooltipHTML;
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
                tooltip.style.opacity = 1;
              }
            })
            .on('mousemove', function (event) {
              if (tooltip) {
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
              }
            })
            .on('mouseout', function () {
              d3.select(this).attr('fill-opacity', 0.85).attr('stroke-width', 0.5);
              if (tooltip) {
                tooltip.style.opacity = 0;
              }
            });

          drawPrideGradientLegend(domainMin, domainMax, colorScale);

          console.log('[MapTab] 六角形網格（模式2）繪製完成');
        } catch (error) {
          console.error('[MapTab] 六角形網格（模式2）繪製失敗:', error);
        }
      };

      /**
       * 🗺️ Pride 六色網格（Grid 模式2）
       */
      const drawHexGridOnly2 = () => {
        if (!g || !hexData2.value || !path) {
          console.error(
            '[MapTab] 無法繪製六角形網格（模式2）: g=',
            !!g,
            'hexData2=',
            !!hexData2.value,
            'path=',
            !!path
          );
          return;
        }

        try {
          console.log('[MapTab] 開始繪製六角形網格（Grid 模式2）');

          g.selectAll('.hex-grid').remove();
          g.selectAll('.county').remove();

          const features = Array.isArray(hexData2.value.features) ? hexData2.value.features : [];
          const valueKey = '有偶_相同性別_總計';
          const getValue = (feature) => Number(feature?.properties?.[valueKey]) || 0;

          const { min, max } = hexData2Stats.value || {};
          const domainMin = Number.isFinite(min) && min > 0 ? min : 0;
          const domainMax = Number.isFinite(max) && max > 0 ? max : domainMin > 0 ? domainMin : 1;
          const colorScale = createPrideGradientScale(domainMin, domainMax);

          const validFeatures = features.filter((feature) => getValue(feature) > 0);

          if (validFeatures.length === 0) {
            console.warn('[MapTab] Grid 模式2 無可用的 Pride 六角格');
            g.selectAll('.legend').remove();
            return;
          }

          const sortedHexes = [...validFeatures].sort((a, b) => getValue(a) - getValue(b));

          console.log('[DEBUG] Grid 模式2 - 有效六角格數量:', sortedHexes.length);
          console.log('[DEBUG] Grid 模式2 - 數值範圍:', { min: domainMin, max: domainMax });

          const hexPaths = g
            .selectAll('.hex-grid')
            .data(sortedHexes)
            .enter()
            .append('path')
            .attr('d', path)
            .attr('class', 'hex-grid')
            .attr('fill', (d) => colorScale(Math.min(domainMax, Math.max(domainMin, getValue(d)))))
            .attr('fill-opacity', 0.85)
            .attr('stroke', '#ffffff')
            .attr('stroke-width', 0.5)
            .style('cursor', 'pointer');

          hexPaths
            .on('mouseover', function (event, d) {
              d3.select(this).attr('fill-opacity', 1).attr('stroke-width', 2);
              if (tooltip) {
                const properties = d.properties || {};
                const formatValue = (key, value) => {
                  if (value === null || value === undefined) return 'N/A';
                  if (typeof value === 'number') {
                    return value.toLocaleString();
                  }
                  return value;
                };
                let tooltipHTML = '';
                Object.keys(properties).forEach((key) => {
                  const value = properties[key];
                  tooltipHTML += `<div><strong>${key}:</strong> ${formatValue(key, value)}</div>`;
                });
                tooltip.innerHTML = tooltipHTML;
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
                tooltip.style.opacity = 1;
              }
            })
            .on('mousemove', function (event) {
              if (tooltip) {
                const [mouseX, mouseY] = d3.pointer(event, mapContainer.value);
                tooltip.style.left = mouseX + 10 + 'px';
                tooltip.style.top = mouseY - 10 + 'px';
              }
            })
            .on('mouseout', function () {
              d3.select(this).attr('fill-opacity', 0.85).attr('stroke-width', 0.5);
              if (tooltip) {
                tooltip.style.opacity = 0;
              }
            });

          drawPrideGradientLegend(domainMin, domainMax, colorScale);

          console.log('[MapTab] 六角形網格（Grid 模式2）繪製完成');
        } catch (error) {
          console.error('[MapTab] 六角形網格（Grid 模式2）繪製失敗:', error);
        }
      };

      /**
       * 🏗️ 創建地圖實例
       * 初始化 D3.js 地圖並設定基本配置
       */
      const createMap = () => {
        if (!mapContainer.value) return false;

        const rect = mapContainer.value.getBoundingClientRect();
        if (rect.width === 0 || rect.height === 0) {
          console.warn('[MapTab] 容器尺寸為零，延遲初始化');
          return false;
        }

        try {
          const width = rect.width;
          const height = rect.height;

          // 台灣中心位置：緯度 23.5°, 經度 121°

          // 創建 SVG 元素
          svg = d3
            .select(mapContainer.value)
            .append('svg')
            .attr('width', width)
            .attr('height', height)
            .style('background', '#ffffff'); // 白色背景

          // 創建投影 - 麥卡托投影，聚焦在台灣
          projection = d3
            .geoMercator()
            .center([121, 23.5]) // 中心點在台灣
            .scale(12000) // 更大的縮放比例，更聚焦在台灣
            .translate([width / 2, height / 2]);

          // 創建路徑生成器
          path = d3.geoPath().projection(projection);

          // 創建容器組
          g = svg.append('g');

          // 設置縮放行為
          zoom = d3
            .zoom()
            .scaleExtent([0.5, 50]) // 允許縮放 0.5x 到 50x
            .on('zoom', (event) => {
              if (g && g.node() && g.node().parentNode) {
                g.attr('transform', event.transform);
              }
            });

          svg.call(zoom);

          // 重置縮放狀態，確保切換模式時不會受到之前模式的影響
          svg.call(zoom.transform, d3.zoomIdentity);

          // 創建工具提示元素
          createTooltip();

          isMapReady.value = true;

          // 將地圖實例傳遞給父組件
          emit('map-ready', { svg, projection, path });

          console.log('[MapTab] D3.js 地圖創建成功');
          return true;
        } catch (error) {
          console.error('[MapTab] D3.js 地圖創建失敗:', error);
          return false;
        }
      };

      /**
       * 🚀 初始化地圖
       * 根據初始顯示模式創建對應的視圖
       */
      const initMap = async () => {
        let attempts = 0;
        const maxAttempts = 20;

        // 根據顯示模式載入不同的數據
        if (displayMode.value === 'map') {
          // 地圖模式：需要載入縣市界線和六角形網格數據
          console.log('[MapTab] 開始載入地圖模式數據...');
          const [countyLoaded, hexLoaded] = await Promise.all([loadCountyData(), loadHexData()]);

          if (!countyLoaded) {
            console.error('[MapTab] 無法載入直轄市、縣(市)界線數據');
            return;
          }

          if (!hexLoaded) {
            console.error('[MapTab] 無法載入六角形網格數據');
            return;
          }

          console.log('[MapTab] 所有數據載入完成，開始創建地圖');

          const tryCreateMap = async () => {
            if (attempts >= maxAttempts) {
              console.error('[MapTab] 地圖初始化失敗，已達到最大嘗試次數');
              return;
            }

            attempts++;
            console.log(`[MapTab] 嘗試創建地圖 (${attempts}/${maxAttempts})`);

            if (createMap()) {
              console.log('[MapTab] 地圖創建成功，開始繪製圖層');
              // 先繪製縣市界線（底層）
              drawCounties();
              // 再繪製六角形網格（上層）
              drawHexGrid();
            } else {
              console.log('[MapTab] 地圖創建失敗，100ms 後重試');
              setTimeout(tryCreateMap, 100);
            }
          };

          tryCreateMap();
        } else if (displayMode.value === 'grid') {
          // Grid 模式：需要載入六角形網格數據，需要地圖投影來繪製
          console.log('[MapTab] 開始載入網格模式數據...');
          const hexLoaded = await loadHexData();

          if (!hexLoaded) {
            console.error('[MapTab] 無法載入六角形網格數據');
            return;
          }

          console.log('[MapTab] 數據載入完成，開始創建網格視圖');

          const tryCreateGrid = async () => {
            if (attempts >= maxAttempts) {
              console.error('[MapTab] 網格初始化失敗，已達到最大嘗試次數');
              return;
            }

            attempts++;
            console.log(`[MapTab] 嘗試創建網格視圖 (${attempts}/${maxAttempts})`);

            if (createMap()) {
              console.log('[MapTab] 網格視圖創建成功，開始繪製六角形網格');
              drawHexGridOnly();
            } else {
              console.log('[MapTab] 網格視圖創建失敗，100ms 後重試');
              setTimeout(tryCreateGrid, 100);
            }
          };

          tryCreateGrid();
        } else if (displayMode.value === 'cesium3d') {
          // CesiumJS 3D 模式
          console.log('[MapTab] 開始載入 CesiumJS 3D 模式數據...');
          const [hexLoaded, countyLoaded] = await Promise.all([loadHexData(), loadCountyData()]);

          if (!hexLoaded) {
            console.error('[MapTab] 無法載入六角形網格數據');
            return;
          }

          if (!countyLoaded) {
            console.error('[MapTab] 無法載入縣市界線數據');
            return;
          }

          console.log('[MapTab] 數據載入完成，開始初始化 CesiumJS 3D 地圖');
          cleanupOtherViews();
          await initCesium3D();
        } else if (displayMode.value === 'maplibre3d') {
          // MapLibre 3D 模式
          console.log('[MapTab] 開始載入 MapLibre 3D 模式數據...');
          const [hexLoaded, countyLoaded] = await Promise.all([loadHexData(), loadCountyData()]);

          if (!hexLoaded) {
            console.error('[MapTab] 無法載入六角形網格數據');
            return;
          }

          if (!countyLoaded) {
            console.error('[MapTab] 無法載入縣市界線數據');
            return;
          }

          console.log('[MapTab] 數據載入完成，開始初始化 MapLibre 3D 地圖');
          cleanupOtherViews();
          await initMapLibre3D();
        }
      };

      // 處理窗口大小調整（重新繪製整個地圖）
      let resizeTimer = null;
      const handleResize = () => {
        // 防抖處理，避免頻繁重繪
        if (resizeTimer) {
          clearTimeout(resizeTimer);
        }
        resizeTimer = setTimeout(() => {
          console.log('[MapTab] 窗口大小調整，重新繪製地圖');
          initMap();
        }, 300);
      };

      // 🧹 生命週期：組件掛載
      onMounted(() => {
        nextTick(() => {
          initMap();
        });

        // 監聽窗口大小調整
        window.addEventListener('resize', handleResize);
      });

      // 🧹 生命週期：組件卸載
      onUnmounted(() => {
        // 清除 resize timer
        if (resizeTimer) {
          clearTimeout(resizeTimer);
        }

        // 移除 resize 監聽器
        window.removeEventListener('resize', handleResize);

        if (svg) {
          // 完全移除 zoom 行為
          try {
            if (zoom) {
              svg.on('.zoom', null);
              svg.call(zoom.on('zoom', null));
            }
          } catch (e) {
            console.warn('[MapTab] 移除 zoom 行為時出錯:', e);
          }
          svg.remove();
          svg = null;
          g = null;
          zoom = null;
        }

        // 清理工具提示
        if (tooltip) {
          tooltip.remove();
          tooltip = null;
        }

        // 清理 Cesium Viewer
        if (cesiumViewer) {
          try {
            cesiumViewer.destroy();
          } catch (e) {
            console.warn('[MapTab] 清理 Cesium Viewer 時出錯:', e);
          }
          cesiumViewer = null;
        }

        // 清理 MapLibre Map
        if (maplibreMap) {
          try {
            maplibreMap.remove();
          } catch (e) {
            console.warn('[MapTab] 清理 MapLibre Map 時出錯:', e);
          }
          maplibreMap = null;
        }

        projection = null;
        path = null;
        zoom = null;
        g = null;
        isMapReady.value = false;
      });

      // 📤 返回組件公開的屬性和方法
      return {
        mapContainer,
        mapContainerId,
        displayMode,
        toggleDisplayMode,
      };
    },
  };
</script>

<template>
  <!-- 🗺️ 地圖主容器 -->
  <div id="map-container" class="h-100 w-100 position-relative bg-transparent z-0">
    <!-- 🗺️ Leaflet 地圖容器 -->
    <div :id="mapContainerId" ref="mapContainer" class="h-100 w-100"></div>

    <!-- 🎛️ 左側中間控制面板 -->
    <div
      class="position-absolute"
      style="top: 50%; left: 0; transform: translateY(-50%); z-index: 1000; padding: 1rem"
    >
      <div class="bg-dark bg-opacity-75 rounded-3 p-3">
        <!-- 🎛️ 顯示模式選擇區域 -->
        <div class="">
          <div class="d-flex flex-column gap-1">
            <button
              type="button"
              class="btn border-0 my-country-btn my-font-sm-white px-4 py-3"
              :class="[displayMode === 'map' ? 'active' : '']"
              @click="toggleDisplayMode('map')"
            >
              地圖模式
            </button>
            <button
              type="button"
              class="btn border-0 my-country-btn my-font-sm-white px-4 py-3"
              :class="[displayMode === 'grid' ? 'active' : '']"
              @click="toggleDisplayMode('grid')"
            >
              網格模式
            </button>
            <button
              type="button"
              class="btn border-0 my-country-btn my-font-sm-white px-4 py-3"
              :class="[displayMode === 'cesium3d' ? 'active' : '']"
              @click="toggleDisplayMode('cesium3d')"
            >
              CesiumJS 3D模式
            </button>
            <button
              type="button"
              class="btn border-0 my-country-btn my-font-sm-white px-4 py-3"
              :class="[displayMode === 'maplibre3d' ? 'active' : '']"
              @click="toggleDisplayMode('maplibre3d')"
            >
              MapLibre 3D模式
            </button>
            <button
              type="button"
              class="btn border-0 my-country-btn my-font-sm-white px-4 py-3"
              :class="[displayMode === 'map2' ? 'active' : '']"
              @click="toggleDisplayMode('map2')"
            >
              地圖模式2
            </button>
            <button
              type="button"
              class="btn border-0 my-country-btn my-font-sm-white px-4 py-3"
              :class="[displayMode === 'grid2' ? 'active' : '']"
              @click="toggleDisplayMode('grid2')"
            >
              網格模式2
            </button>
            <button
              type="button"
              class="btn border-0 my-country-btn my-font-sm-white px-4 py-3"
              :class="[displayMode === 'cesium3d2' ? 'active' : '']"
              @click="toggleDisplayMode('cesium3d2')"
            >
              CesiumJS 3D模式2
            </button>
            <button
              type="button"
              class="btn border-0 my-country-btn my-font-sm-white px-4 py-3"
              :class="[displayMode === 'maplibre3d2' ? 'active' : '']"
              @click="toggleDisplayMode('maplibre3d2')"
            >
              MapLibre 3D模式2
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
  @import '../assets/css/common.css';

  #map-container {
    overflow: hidden;
  }

  /* CesiumJS 容器樣式 */
  :deep(.cesium-viewer) {
    width: 100%;
    height: 100%;
  }

  /* MapLibre GL 容器樣式 */
  :deep(.maplibregl-map) {
    width: 100%;
    height: 100%;
  }

  :deep(.maplibregl-popup-content) {
    background-color: rgba(0, 43, 127, 0.95);
    color: #ffc61e;
    border: 2px solid #ffc61e;
    padding: 10px;
    border-radius: 4px;
  }

  :deep(.leaflet-container) {
    background: #ffffff; /* 白色背景 */
  }

  :deep(.leaflet-popup-content-wrapper) {
    background: rgba(0, 43, 127, 0.95); /* 諾魯深藍色半透明 */
    color: #ffc61e; /* 金黃色文字 */
    border: 2px solid #ffc61e; /* 金黃色邊框 */
  }

  :deep(.leaflet-popup-tip) {
    background: rgba(0, 43, 127, 0.95); /* 諾魯深藍色半透明 */
  }

  :deep(.leaflet-tooltip) {
    background-color: rgba(0, 43, 127, 0.95) !important; /* 諾魯深藍色 */
    color: #ffc61e !important; /* 金黃色文字 */
    border: 1px solid #ffc61e !important; /* 金黃色邊框 */
    font-size: 14px;
    padding: 8px 12px;
    border-radius: 4px;
    line-height: 1.4;
  }

  :deep(.map-tooltip) {
    background-color: #333; /* 深灰色背景 */
    color: #fff; /* 白色文字 */
    border: none; /* 無邊框 */
  }
</style>

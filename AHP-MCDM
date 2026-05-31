// STEP 1: INITIALIZATION & ROI SETUP
var roi = ee.FeatureCollection("users/himangshub167/Districts__Assam");
var crs = 'EPSG:32646'; 
var exportScale = 30; 
var region = roi.geometry();
Map.centerObject(roi, 7);

// STEP 2: ELEVATION (RELIEF) PARAMETER
var srtm = ee.Image("USGS/SRTMGL1_003").clip(roi);

// Scoring Elevation: Lower elevation = higher flood risk
var elevationS = ee.Image(1)
  .where(srtm.lte(100), 5)
  .where(srtm.gt(200).and(srtm.lte(200)), 4)
  .where(srtm.gt(400).and(srtm.lte(400)), 3)
  .where(srtm.gt(700).and(srtm.lte(700)), 2)
  .where(srtm.gt(1000), 1).clip(roi).rename('Elevation_Score');

// STEP 3: TOPOGRAPHIC WETNESS INDEX (TWI) & DRAINAGE DENSITY
var slopeDegrees = ee.Terrain.slope(srtm);
var flowAcc = ee.Image("WWF/HydroSHEDS/15ACC").clip(roi);
var slopeRadians = slopeDegrees.multiply(Math.PI / 180);
var slopeTan = slopeRadians.tan().max(0.001);
var twi = flowAcc.multiply(ee.Image.pixelArea()).divide(slopeTan).log().rename('TWI');

var streamThreshold = 100; 
var streams = flowAcc.gt(streamThreshold);
var kernelRadius = 2000; 
var circularKernel = ee.Kernel.circle(kernelRadius, 'meters', false);
var circleAreaSqKm = Math.PI * Math.pow(kernelRadius / 1000, 2);
var dd = streams.reduceNeighborhood({
  reducer: ee.Reducer.sum(),
  kernel: circularKernel
}).divide(circleAreaSqKm).rename('DD').clip(roi);

// STEP 4: DISTANCE TO RIVER
var jrcWater = ee.Image("JRC/GSW1_4/GlobalSurfaceWater").select('occurrence').clip(roi);
var dist = jrcWater.gt(50).fastDistanceTransform().sqrt().multiply(ee.Image.pixelArea().sqrt()).rename('Dist');

// STEP 5: RAINFALL
var rf = ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY')
  .filterDate('2016-01-01', '2026-01-01')
  .sum().divide(10).rename('RF');

// STEP 6: LAND USE / LAND COVER (LULC)
var lulc = ee.ImageCollection("ESA/WorldCover/v100").first().clip(roi).rename('LULC');

// STEP 7: HISTORICAL FLOOD INUNDATION & EXTENT COMPOSITE
var floodComposite = ee.Image("projects/proven-impact-465305-p7/assets/Flood_Composite").clip(roi);

// Scoring Flood Extent: Areas with historical flooding get max risk score (5), others get (1)
var floodExtentS = ee.Image(1).where(floodComposite.gt(0), 5).clip(roi).rename('FloodExtent_Score');

// STEP 8: RECLASSIFICATION & SCORING (1 to 5 SCALE)
var distS = ee.Image(1).where(dist.lte(1000), 5).where(dist.gt(1000).and(dist.lte(2500)), 4).where(dist.gt(2500).and(dist.lte(5000)), 3).where(dist.gt(5000).and(dist.lte(10000)), 2).where(dist.gt(10000), 1).clip(roi);
var twiS = ee.Image(1).where(twi.gt(15), 5).where(twi.gt(12).and(twi.lte(15)), 4).where(twi.gt(9).and(twi.lte(12)), 3).where(twi.gt(6).and(twi.lte(9)), 2).where(twi.lte(6), 1).clip(roi);
var rfS = ee.Image(1).where(rf.gt(2800), 5).where(rf.gt(2400).and(rf.lte(2800)), 4).where(rf.gt(2000).and(rf.lte(2400)), 3).where(rf.gt(1600).and(rf.lte(2000)), 2).where(rf.lte(1600), 1).clip(roi);
var lulcS = lulc.remap([10, 20, 30, 40, 50, 60, 80, 90], [1, 2, 2, 4, 5, 3, 3, 3], 1).clip(roi);
var ddS = ee.Image(1).where(dd.gt(1.5), 5).where(dd.gt(1.0).and(dd.lte(1.5)), 4).where(dd.gt(0.5).and(dd.lte(1.0)), 3).where(dd.gt(0.2).and(dd.lte(0.5)), 2).where(dd.lte(0.2), 1).clip(roi);

// STEP 9: AHP WEIGHTED OVERLAY CALCULATION

var floodIndex = distS.multiply(0.35)       
                      .add(ddS.multiply(0.10)) 
                      .add(twiS.multiply(0.10))
                      .add(rfS.multiply(0.15)) 
                      .add(lulcS.multiply(0.10)) 
                      .add(floodExtentS.multiply(0.10)) 
                      .add(elevationS.multiply(0.10)) 
                      .rename('Final_Flood_Risk');

// STEP 10: MAP VISUALIZATIONS
Map.addLayer(distS, {min: 1, max: 5, palette: ['#e0f3f8','#abd9e9','#74add1','#4575b4','#313695']}, 'Dist_River_Score', false);
Map.addLayer(elevationS, {min: 1, max: 5, palette: ['#1a9850','#91cf60','#d9ef8b','#fee08b','#fc8d59']}, 'Elevation_Score', false);
Map.addLayer(flood_16_20, {min: 0, max: 1, palette: ['white', 'blue']}, 'Inundation 2016-2020', false);
Map.addLayer(flood_21_25, {min: 0, max: 1, palette: ['white', 'cyan']}, 'Inundation 2021-2025', false);
Map.addLayer(floodIndex, {min: 1, max: 5, palette: ['#2b83ba', '#abdda4', '#ffffbf', '#fdae61', '#d7191c']}, 'Final_AHP_Risk', true);

// STEP 11: EXPORTING LAYERS (.tif to Drive)
var exportLayers = [
  {img: distS, name: 'Dist_River_Score'}, 
  {img: twiS, name: 'TWI_Score'}, 
  {img: ddS, name: 'DD_Score'},
  {img: elevationS, name: 'Elevation_Score'}, 
  {img: rfS, name: 'Rainfall_Score'}, 
  {img: lulcS, name: 'LULC_Score'}, 
  {img: floodIndex, name: 'Final_AHP_Risk'},
  {img: flood_16_20, name: 'Flood_Inundation_2016_2020'}, 
  {img: flood_21_25, name: 'Flood_Inundation_2021_2025'}
];

exportLayers.forEach(function(item) {
  Export.image.toDrive({
    image: item.img.clip(roi),
    description: 'Assam_' + item.name,
    folder: 'Assam_Flood',
    scale: exportScale,
    region: region,
    crs: crs,
    maxPixels: 1e13
  });
});

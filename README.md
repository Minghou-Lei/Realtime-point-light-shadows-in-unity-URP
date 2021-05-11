# Realtime-point-light-shadows-in-unity-URP

Status: In progress

![Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled.png](Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled.png)

unity 2019的URP是不支持实时点光源阴影的，我们可以通过修改urp管线来实现:

![Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/ShadowMap.png](Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/ShadowMap.png)

首先我们先要修改UniversalRenderPipeline.cs这个脚本，这个脚本用C#定义了每一帧渲染管线的执行步骤。我们需要修改InitializeRenderingData这个函数，它的作用是在渲染下一帧前准备好所需的数据：

```csharp
// UniversalRP doesn't support additional directional lights or point light shadows yet
// 遇到下述光照类型时才考虑计算光照，我们要把点光也给加上：
if (visibleLights[i].lightType == LightType.Spot && light != null && light.shadows != LightShadows.None){
    additionalLightsCastShadows = true;
    break;
}
```

修改成：

```csharp
// 加入点光的情况，也让他计算阴影
if ( (visibleLights[i].lightType == LightType.Spot || visibleLights[i].lightType == LightType.Point) && light != null && light.shadows != LightShadows.None){
    additionalLightsCastShadows = true;
    break;
}
```

接下来，如果判断additionalLightsCastShadows为真 →就会进入到AdditionalLightsShadowCasterPass.cs这个脚本，进行ShadowMap 图集的绘制，我们可以仿照Spot Light的写法，使用ShadowUtils.cs绘制ShadowMap并进行图集的Index管理:

Spot Light ShadowMap绘制的写法(Setup阶段):

```csharp
if (lightType == LightType.Spot)
{
// 提取Spot Light的view matrix,project matrix，获得ShadowTransform
    bool success = ShadowUtils.ExtractSpotLightMatrix(ref renderingData.cullResults,
        ref renderingData.shadowData,
        globalLightIndex,
        out var shadowTransform,
        out m_AdditionalLightsShadowSlices[globalShadowSliceIndex].viewMatrix,
        out m_AdditionalLightsShadowSlices[globalShadowSliceIndex].projectionMatrix);

    if (success)
    {
// 确定这个光源会造成阴影，加入待渲染ShadowMap的列表
        m_ShadowSliceToGlobalLightIndex.Add(globalLightIndex);
        var light = shadowLight.light;
        float shadowStrength = light.shadowStrength;
// 设置好阴影参数 -> [阴影强度，是否启用Soft-Shadows，光源类型（Spot？Point？），Index偏置]
        float softShadows = (supportsSoftShadows && light.shadows == LightShadows.Soft) ? 1.0f : 0.0f;
        Vector4 shadowParams = new Vector4(shadowStrength, softShadows, LightTypeIdentifierInShadowParams_Spot, perLightFirstShadowSliceIndex);
        if (m_UseStructuredBuffer)
        {
            m_AdditionalLightsShadowData[globalShadowSliceIndex].worldToShadowMatrix = shadowTransform;
            m_AdditionalLightsShadowData[globalShadowSliceIndex].shadowParams = shadowParams;
        }
        else
        {
// 保存 图集中该ShadowMap的Index -> WorldShadowMatrix的映射关系
            m_AdditionalLightShadowSliceIndexTo_WorldShadowMatrix[globalShadowSliceIndex] = shadowTransform;
// 保存 GlobalLightIndex -> ShadowedAdditionalLightIndex的映射关系
            m_GlobalLightIndexToShadowedAdditionalLightIndex[globalLightIndex] = validShadowCastingLightsCount;
// 保存该Index的阴影参数
            m_ShadowedAdditionalLightIndexToShadowParams[validShadowCastingLightsCount] = shadowParams;
        }
// 确定渲染该ShadowMap（没有被剔除）
        isValidShadowSlice = true;
// 确定这个光源会造成阴影
        isValidShadowCastingLight = true;
    }
}
```

不同于Spot Light的是，Point Light需要取样6次（上下左右前后），而Spot Light只需要取样一次，所以当判断 lightType == LightType.Point 时，加入6次ShadowMap的渲染请求：

```csharp
private int GetPunctualLightShadowSlicesCount(in LightType lightType)
{
    switch (lightType)
    {
        case LightType.Spot:
            return 1;
        case LightType.Point:
            return 6;
        default:
            return 0;
    }
}
```

根据光源种类进行迭代

```csharp
for (int globalLightIndex = 0; globalLightIndex < visibleLights.Length && m_ShadowSliceToGlobalLightIndex.Count < totalShadowSlicesCount; ++globalLightIndex)
{
    VisibleLight shadowLight = visibleLights[globalLightIndex];

    LightType lightType = shadowLight.lightType;
// 根据光源种类进行迭代
    int perLightShadowSlicesCount = GetPunctualLightShadowSlicesCount(lightType);

    int perLightFirstShadowSliceIndex = m_ShadowSliceToGlobalLightIndex.Count; // shadowSliceIndex within the global array of all shadowed additional lights

    for (int perLightShadowSlice = 0; perLightShadowSlice < perLightShadowSlicesCount; ++perLightShadowSlice)
    {
        int globalShadowSliceIndex = m_ShadowSliceToGlobalLightIndex.Count; // shadowSliceIndex within the global array of all shadowed additional lights

            
                if (lightType == LightType.Spot)
                {
························
```

有了上面的基础，我们就可以写采样Point Light CubeMap的函数了：

```csharp
else if (lightType == LightType.Point)
{
    float fovBias = GetPointLightShadowFrustumFovBiasInDegrees(sliceResolution, (shadowLight.light.shadows == LightShadows.Soft));

    // store fovBias in spotAngle because it is used to compute ShadowUtils.GetShadowBias
    shadowLight.spotAngle = 90 + fovBias;
    bool success = ShadowUtils.ExtractPointLightMatrix(ref renderingData.cullResults,
        ref renderingData.shadowData,
        globalLightIndex,
        (CubemapFace)perLightShadowSlice,
        fovBias/* fovBias in degrees (adds to the 90 deg. of a face frustum fov, avoids missing shadows at faces boundaries) - HDRP calls HDShadowUtils.CalcGuardAnglePerspective to compute a precise value */,
        out var shadowTransform,
        out m_AdditionalLightsShadowSlices[globalShadowSliceIndex].viewMatrix,
        out m_AdditionalLightsShadowSlices[globalShadowSliceIndex].projectionMatrix);

    if (success)
    {
        m_ShadowSliceToGlobalLightIndex.Add(globalLightIndex);
        var light = shadowLight.light;
        float shadowStrength = light.shadowStrength;
        float softShadows = (supportsSoftShadows && light.shadows == LightShadows.Soft) ? 1.0f : 0.0f;
        Vector4 shadowParams = new Vector4(shadowStrength, softShadows, LightTypeIdentifierInShadowParams_Point, perLightFirstShadowSliceIndex);
        if (m_UseStructuredBuffer)
        {
            m_AdditionalLightsShadowData[globalShadowSliceIndex].worldToShadowMatrix = shadowTransform;
            m_AdditionalLightsShadowData[globalShadowSliceIndex].shadowParams = shadowParams;
        }
        else
        {
            m_AdditionalLightShadowSliceIndexTo_WorldShadowMatrix[globalShadowSliceIndex] = shadowTransform;
            m_GlobalLightIndexToShadowedAdditionalLightIndex[globalLightIndex] = validShadowCastingLightsCount;
            m_ShadowedAdditionalLightIndexToShadowParams[validShadowCastingLightsCount] = shadowParams;
        }
        isValidShadowSlice = true;
        isValidShadowCastingLight = true;
    }
}
```

其中：

```csharp
public static bool ExtractPointLightMatrix(ref CullingResults cullResults, ref ShadowData shadowData, int shadowLightIndex, CubemapFace cubemapFace, float fovBias, out Matrix4x4 shadowMatrix, out Matrix4x4 viewMatrix, out Matrix4x4 projMatrix)
{
    ShadowSplitData splitData;
    bool success = cullResults.ComputePointShadowMatricesAndCullingPrimitives(shadowLightIndex, cubemapFace, fovBias, out viewMatrix, out projMatrix, out splitData); // returns false if input parameters are incorrect (rare)
                                                                                                                                                                      // In native API CullingResults.ComputeSpotShadowMatricesAndCullingPrimitives there is code that inverts the 3rd component of shadow-casting spot light's "world-to-local" matrix (it was so since its original addition to the code base):
                                                                                                                                                                      // https://github.cds.internal.unity3d.com/unity/unity/commit/34813e063526c4be0ef0448dfaae3a911dd8be58#diff-cf0b417fc6bd8ee2356770797e628cd4R331
                                                                                                                                                                      //
                                                                                                                                                                      // However native API CullingResults.ComputePointShadowMatricesAndCullingPrimitives does not contain this transformation.
                                                                                                                                                                      // As a result, the view matrices returned for a point light shadow face, and for a spot light with same direction as that face, have opposite 3rd component.
                                                                                                                                                                      //
                                                                                                                                                                      // This causes normalBias to be incorrectly applied to shadow caster vertices during the point light shadow pass.
                                                                                                                                                                      // To counter this effect, we invert the point light shadow view matrix component here:
    {
        viewMatrix.m10 = -viewMatrix.m10;
        viewMatrix.m11 = -viewMatrix.m11;
        viewMatrix.m12 = -viewMatrix.m12;
        viewMatrix.m13 = -viewMatrix.m13;
    }
    shadowMatrix = GetShadowTransform(projMatrix, viewMatrix);
    return success;
}
```

![Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%201.png](Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%201.png)

该函数输出的ShadowSliceData，包含图集

[MVP变换矩阵推导及C++实现_湖广午王-CSDN博客_mvp变换](https://blog.csdn.net/junzia/article/details/85939783)

在setup该帧的结尾，会计算每个Slice的Offset值，对他们根据Index进行排序并计算world Matrix：

```cpp
// 从头开始遍历所有待渲染的Shadow Slice
int sliceIndex = 0;
            //int shadowCastingLightsBufferCount = m_ShadowSlicesToAdditionalLightGlobalIndices.Count;
// 设定默认值
            Matrix4x4 sliceTransform = Matrix4x4.identity;
            sliceTransform.m00 = sliceResolution * oneOverAtlasWidth;
            sliceTransform.m11 = sliceResolution * oneOverAtlasHeight;

            for (int globalShadowSliceIndex = 0; globalShadowSliceIndex < shadowCastingLightsBufferCount; ++globalShadowSliceIndex)
            {
                int globalLightIndex = m_ShadowSliceToGlobalLightIndex[globalShadowSliceIndex];
                int shadowedAdditionalLightIndex = m_GlobalLightIndexToShadowedAdditionalLightIndex[globalLightIndex];

                // we can skip the slice if strength is zero. Some slices with zero
                // strength exists when using uniform array path.
                if (!m_UseStructuredBuffer && ( shadowedAdditionalLightIndex==-1 || Mathf.Approximately(m_ShadowedAdditionalLightIndexToShadowParams[shadowedAdditionalLightIndex].x, 0.0f) ))
                    continue;
// 根据图片的Index值来计算Offset
                m_AdditionalLightsShadowSlices[globalShadowSliceIndex].offsetX = (sliceIndex % shadowSlicesPerRow) * sliceResolution;
                m_AdditionalLightsShadowSlices[globalShadowSliceIndex].offsetY = (sliceIndex / shadowSlicesPerRow) * sliceResolution;
                m_AdditionalLightsShadowSlices[globalShadowSliceIndex].resolution = sliceResolution;

                sliceTransform.m03 = m_AdditionalLightsShadowSlices[globalShadowSliceIndex].offsetX * oneOverAtlasWidth;
                sliceTransform.m13 = m_AdditionalLightsShadowSlices[globalShadowSliceIndex].offsetY * oneOverAtlasHeight;

                // We bake scale and bias to each shadow map in the atlas in the matrix.
                // saves some instructions in shader.
                if (m_UseStructuredBuffer)
                {
                    // With the introduction of point light shadows, the number of shadow slices becomes different from the number of punctual lights.
                    // Therefore light data and shadow slice data must be stored in different structured buffers (of different sizes).
                    // TODO: Check and fix "Structured Buffers" code path - Possibly use one SSBO for light data and one for shadow slice data (same as UBO code path)
                    m_AdditionalLightsShadowData[globalShadowSliceIndex].worldToShadowMatrix = sliceTransform * m_AdditionalLightsShadowData[globalShadowSliceIndex].worldToShadowMatrix;
                }
                else
                {
                    m_AdditionalLightShadowSliceIndexTo_WorldShadowMatrix[globalShadowSliceIndex] = sliceTransform * m_AdditionalLightShadowSliceIndexTo_WorldShadowMatrix[globalShadowSliceIndex];
                }
                sliceIndex++;
            }
```

![Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%202.png](Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%202.png)

slice信息

SliceTransform是视口变换矩阵，其中：

![Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%203.png](Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%203.png)

当Index = 0时，SliceTransform = 

![Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%204.png](Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%204.png)

进行了缩放

当Index = 1时，SliceTransform = 

![Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%205.png](Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%205.png)

缩放，并在x轴上平移

当Index = 5时，SliceTransform = 

![Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%206.png](Realtime-point-light-shadows-in-unity-URP%20baf834be833f442c9610cd00f4c22bf7/Untitled%206.png)

缩放，在X轴和Y轴上平移

Shader阶段：

```cpp
Light GetAdditionalPerObjectLight(intperObjectLightIndex, float3 positionWS)
{
// Abstraction over Light input constants
#ifUSE_STRUCTURED_BUFFER_FOR_LIGHT_DATA
    float4 lightPositionWS = _AdditionalLightsBuffer[perObjectLightIndex].position;
    half3 color = _AdditionalLightsBuffer[perObjectLightIndex].color.rgb;
    half4 distanceAndSpotAttenuation = _AdditionalLightsBuffer[perObjectLightIndex].attenuation;
    half4 spotDirection = _AdditionalLightsBuffer[perObjectLightIndex].spotDirection;
    half4 lightOcclusionProbeInfo = _AdditionalLightsBuffer[perObjectLightIndex].occlusionProbeChannels;
#else
float4 lightPositionWS = _AdditionalLightsPosition[perObjectLightIndex];
    half3 color = _AdditionalLightsColor[perObjectLightIndex].rgb;
    half4 distanceAndSpotAttenuation = _AdditionalLightsAttenuation[perObjectLightIndex];
    half4 spotDirection = _AdditionalLightsSpotDir[perObjectLightIndex];
    half4 lightOcclusionProbeInfo = _AdditionalLightsOcclusionProbes[perObjectLightIndex];
#endif

// Directional lights store direction in lightPosition.xyz and have .w set to 0.0.
// This way the following code will work for both directional and punctual lights.
float3 lightVector = lightPositionWS.xyz - positionWS * lightPositionWS.w;
floatdistanceSqr = max(dot(lightVector, lightVector), HALF_MIN);

    half3 lightDirection = half3(lightVector * rsqrt(distanceSqr));
halfattenuation = DistanceAttenuation(distanceSqr, distanceAndSpotAttenuation.xy) * AngleAttenuation(spotDirection.xyz, lightDirection, distanceAndSpotAttenuation.zw);

    Light light;
    light.direction = lightDirection;
    light.distanceAttenuation = attenuation;
    light.shadowAttenuation = AdditionalLightRealtimeShadow(perObjectLightIndex, positionWS, lightDirection);
    light.color = color;

// In case we're using light probes, we can sample the attenuation from the `unity_ProbesOcclusion`
#ifdefined(LIGHTMAP_ON) || defined(_MIXED_LIGHTING_SUBTRACTIVE)
// First find the probe channel from the light.
// Then sample `unity_ProbesOcclusion` for the baked occlusion.
// If the light is not baked, the channel is -1, and we need to apply no occlusion.

// probeChannel is the index in 'unity_ProbesOcclusion' that holds the proper occlusion value.
intprobeChannel = lightOcclusionProbeInfo.x;

// lightProbeContribution is set to 0 if we are indeed using a probe, otherwise set to 1.
half lightProbeContribution = lightOcclusionProbeInfo.y;

    half probeOcclusionValue = unity_ProbesOcclusion[probeChannel];
    light.distanceAttenuation *= max(probeOcclusionValue, lightProbeContribution);
#endif

returnlight;
}
```

```cpp
halfAdditionalLightRealtimeShadow(intlightIndex, float3 positionWS, half3 lightDirection)
{
#if!defined(ADDITIONAL_LIGHT_CALCULATE_SHADOWS)
return1.0h;
#endif

ShadowSamplingData shadowSamplingData = GetAdditionalLightShadowSamplingData();

#ifUSE_STRUCTURED_BUFFER_FOR_LIGHT_DATA

// With the introduction of point light shadows, the number of shadow slices becomes different from the number of punctual lights.
    // Therefore light data and shadow slice data must be stored in different structured buffers (of different sizes).
    // TODO: check and fix "Structured Buffers" code path - Possibly use one SSBO for light data and one for shadow slice data (same as UBO code path)

lightIndex = _AdditionalShadowsIndices[lightIndex];// shadow slice index

    // We have to branch here as otherwise we would sample buffer with lightIndex == -1.
    // However this should be ok for platforms that store light in SSBO.
UNITY_BRANCH
if(lightIndex < 0)
return1.0;

    float4 shadowCoord = mul(_AdditionalShadowsBuffer[lightIndex].worldToShadowMatrix, float4(positionWS, 1.0));

#else

half4 shadowParams = GetAdditionalLightShadowParams(lightIndex);

intshadowSliceIndex = shadowParams.w;

if(shadowParams.z)
    {
// 对于屏幕上的每一点像素，计算到光源的向量，获得CubeMap中所朝向的面
floatcubemapFaceId = CubeMapFaceID(-lightDirection);
        shadowSliceIndex += cubemapFaceId;
    }
    float4 shadowCoord = mul(_AdditionalLightsWorldToShadow[shadowSliceIndex], float4(positionWS, 1.0));

#endif

    returnSampleShadowmap(TEXTURE2D_ARGS(_AdditionalLightsShadowmapTexture, sampler_AdditionalLightsShadowmapTexture), shadowCoord, shadowSamplingData, shadowParams,true);
}

```

```cpp
real SampleShadowmap(TEXTURE2D_SHADOW_PARAM(ShadowMap, sampler_ShadowMap), float4 shadowCoord, ShadowSamplingData samplingData, half4 shadowParams, bool isPerspectiveProjection = true)
{
    // Compiler will optimize this branch away as long as isPerspectiveProjection is known at compile time
    if (isPerspectiveProjection)
        shadowCoord.xyz /= shadowCoord.w;

    real attenuation;
    real shadowStrength = shadowParams.x;

    // TODO: We could branch on if this light has soft shadows (shadowParams.y) to save perf on some platforms.
#ifdef _SHADOWS_SOFT
    attenuation = SampleShadowmapFiltered(TEXTURE2D_SHADOW_ARGS(ShadowMap, sampler_ShadowMap), shadowCoord, samplingData);
#else
    // 1-tap hardware comparison
    attenuation = SAMPLE_TEXTURE2D_SHADOW(ShadowMap, sampler_ShadowMap, shadowCoord.xyz);
#endif

    attenuation = LerpWhiteTo(attenuation, shadowStrength);

    // Shadow coords that fall out of the light frustum volume must always return attenuation 1.0
    // TODO: We could use branch here to save some perf on some platforms.
    return BEYOND_SHADOW_FAR(shadowCoord) ? 1.0 : attenuation;
}
```
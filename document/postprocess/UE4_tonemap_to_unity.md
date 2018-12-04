## UE4的Film ACES移植到unity中

### 位置

unity端位于Colors.hlsl AcesTonemap

UE4端位于TonemapCommon.hlsl FilmToneMap

### ACES 颜色空间

- [ACES2065-1](https://en.wikipedia.org/wiki/Academy_Color_Encoding_System#ACES2065-1)
- [ACEScg](https://en.wikipedia.org/wiki/Academy_Color_Encoding_System#ACEScg)
- [ACEScc](https://en.wikipedia.org/wiki/Academy_Color_Encoding_System#ACEScc)
- [Converting ACES2065-1 RGB values to CIE *XYZ* values](https://en.wikipedia.org/wiki/Academy_Color_Encoding_System#Converting_ACES2065-1_RGB_values_to_CIE_XYZ_values)
- [Converting CIE *XYZ* values to ACES2065-1 values](https://en.wikipedia.org/wiki/Academy_Color_Encoding_System#Converting_CIE_XYZ_values_to_ACES2065-1_values)

### unity-UE4变量对应

| 变量，颜色空间 | unity    | UE4          |
| -------------- | -------- | ------------ |
| aces           | aces     | ColorAP0     |
| acescg         | acescg   | WorkingColor |
| API1           | linearCV | ToneColor    |

可以知道二者都是在acescg color space进行亮度映射

增加变量

```
float FilmSlope;// = 0.91;
float FilmToe;// = 0.53;
float FilmShoulder;// = 0.23;
float FilmBlackClip;// = 0;
float FilmWhiteClip;// = 0.035;
```

对unity AcesTonamp如下字段进行修改

```c
const float a = 278.5085;
const float b = 10.7772;
const float c = 293.6045;
const float d = 88.7122;
const float e = 80.6889;
float3 x = acescg;
float3 rgbPost = (x * (a * x + b)) / (x * (c * x + d) + e);
float3 linearCV = darkSurround_to_dimSurround(rgbPost);
```

```c
#ifdef UE4_TONEMAP
    const half ToeScale = 1 + FilmBlackClip - FilmToe;
    const half ShoulderScale = 1 + FilmWhiteClip - FilmShoulder;
    const float InMatch = 0.18;
    const float OutMatch = 0.18;
    float ToeMatch;
    if (FilmToe > 0.8)
    {
        // 0.18 will be on straight segment
        ToeMatch = (1 - FilmToe - OutMatch) / FilmSlope + log10(InMatch);
    }
    else
    {
        // 0.18 will be on toe segment
        // Solve for ToeMatch such that input of InMatch gives output of OutMatch.
        const float bt = (OutMatch + FilmBlackClip) / ToeScale - 1;
        ToeMatch = log10(InMatch) - 0.5 * log((1 + bt) / (1 - bt)) * (ToeScale / FilmSlope);
    }

    float StraightMatch = (1 - FilmToe) / FilmSlope - ToeMatch;
    float ShoulderMatch = FilmShoulder / FilmSlope - StraightMatch;
    half3 LogColor = log10(acescg);
    half3 StraightColor = FilmSlope * (LogColor + StraightMatch);
    half3 ToeColor = (-FilmBlackClip) + (2 * ToeScale) / (1 + exp((-2 * FilmSlope / ToeScale) * (LogColor - ToeMatch)));
    half3 ShoulderColor = (1 + FilmWhiteClip) - (2 * ShoulderScale) / (1 + exp((2 * FilmSlope / ShoulderScale) * (LogColor - ShoulderMatch)));
    ToeColor = LogColor < ToeMatch ? ToeColor : StraightColor;
    ShoulderColor = LogColor > ShoulderMatch ? ShoulderColor : StraightColor;
    half3 t = saturate((LogColor - ToeMatch) / (ShoulderMatch - ToeMatch));
    t = ShoulderMatch < ToeMatch ? 1 - t : t;
    t = (3 - 2 * t)*t*t;
    half3 linearCV = lerp(ToeColor, ShoulderColor, t);
    linearCV = lerp(dot(float3(linearCV), AP1_RGB2Y), linearCV, 0.93);
    linearCV = max(0, linearCV);
#else
    const float a = 278.5085;
    const float b = 10.7772;
    const float c = 293.6045;
    const float d = 88.7122;
    const float e = 80.6889;
    float3 x = acescg;
    float3 rgbPost = (x * (a * x + b)) / (x * (c * x + d) + e);
    float3 linearCV = darkSurround_to_dimSurround(rgbPost);
#endif
```

修改ColorGrading.cs,ColorGradingEditor.cs将FilmSlope、FilmToe、FilmShoulder、FilmBlackClip、FilmWhiteClip暴露出去
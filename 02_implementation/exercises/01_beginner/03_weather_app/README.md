# 天気予報アプリ ⭐

**難易度**: 初級
**推奨時間**: 6-8時間
**技術スタック**: React, TypeScript, Weather API

---

## 📋 概要

外部APIを使って天気情報を取得・表示するアプリを作成します。非同期処理とエラーハンドリングの基礎を学びます。

---

## 🎯 学習目標

- [ ] fetch APIによるデータ取得
- [ ] 非同期処理（async/await）
- [ ] ローディング状態の管理
- [ ] エラーハンドリング
- [ ] API連携の基礎

---

## 📝 機能要件

### 必須機能

1. **都市検索**
   - 都市名で天気を検索
   - 検索ボタン/Enterキーで検索

2. **天気表示**
   - 現在の気温
   - 天気状態（晴れ、曇り、雨など）
   - 天気アイコン
   - 湿度、風速

3. **ローディング表示**
   - データ取得中はローディング表示

4. **エラーハンドリング**
   - 都市が見つからない場合のエラー
   - ネットワークエラーの処理

---

## 💻 実装例

```typescript
interface WeatherData {
  city: string;
  temperature: number;
  description: string;
  humidity: number;
  windSpeed: number;
  icon: string;
}

const fetchWeather = async (city: string): Promise<WeatherData> => {
  const API_KEY = process.env.REACT_APP_WEATHER_API_KEY;
  const response = await fetch(
    `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${API_KEY}&units=metric`
  );

  if (!response.ok) {
    throw new Error('City not found');
  }

  const data = await response.json();
  return {
    city: data.name,
    temperature: data.main.temp,
    description: data.weather[0].description,
    humidity: data.main.humidity,
    windSpeed: data.wind.speed,
    icon: data.weather[0].icon,
  };
};
```

---

## 📚 APIの設定

1. [OpenWeatherMap](https://openweathermap.org/api) でアカウント作成
2. API Keyを取得
3. `.env`ファイルに設定

```bash
REACT_APP_WEATHER_API_KEY=your_api_key_here
```

---

**最終更新**: 2025-10-22

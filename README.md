
# Coinglass Price Change
- Step 1: Calculate n based on the URL path.
- Step 2: Calculate m based on the response header "user" and n.
- Step 3: Use m as the key to decrypt the AES ciphertext "data".
- Step 4: Sort ...

----------------------------------------------------
* Types
```
type (
	CoinGlassPriceChange []CoinItem
	CoinItem             struct {
		H12PriceChangePercent float64 `json:"h12PriceChangePercent"`
		H1PriceChangePercent  float64 `json:"h1PriceChangePercent"`
		H24PriceChangePercent float64 `json:"h24PriceChangePercent"`
		H4PriceChangePercent  float64 `json:"h4PriceChangePercent"`
		M15PriceChangePercent float64 `json:"m15PriceChangePercent"`
		M30PriceChangePercent float64 `json:"m30PriceChangePercent"`
		M5PriceChangePercent  float64 `json:"m5PriceChangePercent"`
		Price                 float64 `json:"price"`
		Symbol                string  `json:"symbol"`
		SymbolLogo            string  `json:"symbolLogo,omitempty"`
		VolUsd                float64 `json:"volUsd"`
	}
)

func (i CoinItem) GetValue(key string) float64 {
	switch key {
	case "h1":
		return i.H1PriceChangePercent
	case "m30":
		return i.M30PriceChangePercent
	default:
		return i.H24PriceChangePercent
	}
}
```

* Utils/gzip
```
package gzip

import (
	"bytes"
	"compress/gzip"
	"io"
)

func Compress(data []byte) ([]byte, error) {
	compressedBuffer := bytes.NewReader(data)
	gzipReader, err := gzip.NewReader(compressedBuffer)
	if err != nil {
		return nil, err
	}
	defer gzipReader.Close()
	decompressedBuffer := new(bytes.Buffer)
	_, err = io.Copy(decompressedBuffer, gzipReader)
	if err != nil {
		return nil, err
	}
	return decompressedBuffer.Bytes(), nil
}

```

* Utils/tranform

```
import
	"encoding/base64"
	"github.com/forgoer/openssl"
    "utils/gzip"

func calcN(e string) string {
	n := base64.StdEncoding.EncodeToString([]byte("coinglass" + e + "coinglass"))
	n = n[:16]
	return n
}

func transform(t, e string) ([]byte, error) {
	cipherBytes, err := base64.StdEncoding.DecodeString(t)
	if err != nil {
		return nil, err
	}
	plainBytes, err := openssl.AesECBDecrypt(cipherBytes, []byte(e), openssl.PKCS7_PADDING)
	if err != nil {
		return nil, err
	}
	out, err := gzip.Compress(plainBytes)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

```
const (
	PathURL = "/api/futures/coins/priceChange"
	BaseURL = "https://fapi.coinglass.com"
)

func GetTopGainer(ctx context.Context, key string) {
	r, rH, _, err := client.From(ctx).Get(BaseURL+PathURL+"?ex=Binance", map[string]string{
		"cache-ts": fmt.Sprintf("%d", time.Now().UnixNano()),
	})
	if err != nil {
		return
	}
	user, data := rH.Get("user"), gjson.GetBytes(r, "data").String()
	if lo.IsEmpty(user) || lo.IsEmpty(data) {
		return
	}
	n := calcN(PathURL)
	m, err := transform(user, n)
	if err != nil {
		return
	}
	plainData, err := transform(data, string(m))
	if err != nil {
		return
	}
	var topGainer model.CoinGlassPriceChange
	if err := json.Unmarshal(plainData, &topGainer); err != nil {
		fmt.Println(err)
	}

	newList := lo.Filter(topGainer, func(item model.CoinItem, _ int) bool {
		return item.GetValue(key) > 0
	})

	sort.Slice(newList, func(i, j int) bool {
		return newList[i].GetValue(key) > newList[j].GetValue(key)
	})

	for _, coin := range newList {
		fmt.Printf("Symbol: %s, H1PriceChangePercent: %.2f\n", coin.Symbol, coin.GetValue(key))
	}
}
```
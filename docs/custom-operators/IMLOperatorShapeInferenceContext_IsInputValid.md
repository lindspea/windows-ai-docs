---
author: eliotcowley
title: IMLOperatorShapeInferenceContext.IsInputValid method
description: Returns true if an input to the operator is valid.
ms.author: elcowle
ms.date: 11/1/2018
ms.topic: article
keywords: windows 10, windows machine learning, WinML, custom operators, IsInputValid
ms.localizationpriority: medium
topic_type:
- APIRef
api_type:
- NA
api_name:
- IMLOperatorShapeInferenceContext.IsInputValid
api_location:
- MLOperatorAuthor.h
---

# IMLOperatorShapeInferenceContext.IsInputValid method

Returns true if an input to the operator is valid. This always returns true except for optional inputs and invalid indices.

```cpp
bool IsInputValid(
    uint32_t inputIndex)
```

## Requirements

| | |
|-|-|
| **Minimum supported client** | Windows 10, build 17763 |
| **Header** | MLOperatorAuthor.h |

[!INCLUDE [help](../includes/get-help.md)]

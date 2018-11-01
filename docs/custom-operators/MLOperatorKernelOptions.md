---
author: eliotcowley
title: MLOperatorKernelOptions enum
description: Specifies options used when registering custom operator kernels.
ms.author: elcowle
ms.date: 11/1/2018
ms.topic: article
ms.prod: windows
ms.technology: uwp
keywords: windows 10, windows machine learning, WinML, custom operators, MLOperatorKernelOptions
ms.localizationpriority: medium
topic_type:
- APIRef
api_type:
- NA
api_name:
- MLOperatorKernelOptions
api_location:
- MLOperatorAuthor.h
---

# MLOperatorKernelOptions enum

Specifies options used when registering custom operator kernels.

## Fields

| Name | Value | Description |
|------|-------|-------------|
| None | 0 | |
| AllowDynamicInputShapes | 1 | Specifies whether the shapes of input tensors are allowed to vary among invocations of an operator kernel instance. If this is not set, kernel instances may query input tensor shapes during creation, and front-load initialization work which depends on those shapes. Setting this may improve performance if shapes vary dynamically between inference operations, and the kernel implementation handles this efficiently. |

## Requirements

| | |
|-|-|
| **Minimum supported client** | Windows 10, build 17763 |
| **Header** | MLOperatorAuthor.h |

[!INCLUDE [help](../includes/get-help.md)]

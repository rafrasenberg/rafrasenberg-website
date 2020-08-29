---
title: "Django"
date: 2020-07-11T14:02:25+02:00
draft: true
---

{{< followers >}}

{{< rawhtml >}}
<script src="https://cdnjs.cloudflare.com/ajax/libs/axios/0.19.2/axios.min.js"></script>
 
 <script>
// axios.get('https://j3kxcrii3j.execute-api.eu-central-1.amazonaws.com/default/getFollowerCount', {
// 		mode: 'no-cors',
//     }).then(response => {
//         console.log(response.data);
//     })
//     .catch(error => console.error(error))

fetch('https://j3kxcrii3j.execute-api.eu-central-1.amazonaws.com/default/getFollowerCount')
  .then(response => response.json())
  

 </script>
{{< /rawhtml >}}


{{< rawhtml >}}<div id="follower_count"></div>{{< /rawhtml >}}


:smile:

```javascript
let eternalLove = (male, female) => {
    if (male === "Raf" && female === "Maria") {
        return true;
    } else {
        return false;
    }
}

eternalLove("Raf", "Maria")
```
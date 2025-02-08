---
title: "Tax code calculator according to the Italian specification"
---

This italian tax code calculator is totally offline and does not share your data with any server!

<form id="form-cf"">
  <label for="firstName">First name:</label>
  <input type="text" id="firstName" required value="Sofia">

  <label for="lastName">Last name:</label>
  <input type="text" id="lastName" required value="Rossi">

  <label for="birthDate">Date of birth:</label>
  <input type="date" id="birthDate" required value="1990-01-01">

  <label for="sesso">Sex:</label>
  <input class="button" type="radio" name="sex" id="female" checked/>
  <label for="female">F</label>
  <input class="button" type="radio" name="sex" id="male" />
  <label for="male">M</label> 
  
  <label id="municipalityLabel" for="municipalityCode">Municipality:</label>
  <input type="text" id="municipalityCode" required value="Abano Terme (A001)">
  <ul id="municipalitySuggestions" style="list-style-type: none; padding: 0; margin: 0; display: none;"></ul>

</form>

<h3>Tax code:</h3>
<p id="cf-result"></p>

<script>
  let selectedMunicipalityCode = "A001";

   let comuniData = [];

  async function loadMunicipalityData() {
    const url = 'https://raw.githubusercontent.com/opendatasicilia/comuni-italiani/main/dati/comuni_codici-catastali.csv';
    
    try {
      const response = await fetch(url);
      const csvText = await response.text();
      
      const rows = csvText.split('\n').slice(1);
      comuniData = rows.map(row => {
        const columns = row.split(',');
        if (columns.length != 3) {
          return null;
        }
        
        return {
          codiceCatastale: columns[1].trim(),
          comune: columns[2].trim(),
        };
      }).filter(item => item !== null);

    } catch (error) {
      console.error("Error loadMunicipalityData:", error);
    }
  }

  function findMunicipalityCode(comune) {
    return comuniData.filter(item => item.comune.toLowerCase().includes(comune.toLowerCase())).slice(0, 10);
  }

  function updateSuggestions(matches) {
    const suggestionsList = document.getElementById("municipalitySuggestions");
    suggestionsList.innerHTML = "";
    if (matches.length > 0) {
      matches.forEach(item => {
        const li = document.createElement("li");
        li.textContent = `${item.comune} (${item.codiceCatastale})`;
        li.addEventListener("click", () => selectMunicipality(item));
        suggestionsList.appendChild(li);
      });
      suggestionsList.style.display = "block";
    } else {
      suggestionsList.style.display = "none";
    }
  }

  function selectMunicipality(item) {
    document.getElementById("municipalityCode").value = `${item.comune} (${item.codiceCatastale})`;
    document.getElementById("municipalitySuggestions").style.display = "none";
    selectedMunicipalityCode = item.codiceCatastale;
    calculateTaxCode();
  }

  function calculateTaxCode() {
    const firstName = document.getElementById("firstName").value.trimStart().trimEnd();
    const lastName = document.getElementById("lastName").value.trimStart().trimEnd();
    const birthDate = new Date(document.getElementById("birthDate").value);
    const sex = document.querySelector('input[name="sex"]:checked').id;
    let municipalityCode = document.getElementById("municipalityCode").value;
    
    if (selectedMunicipalityCode) {
      municipalityCode = selectedMunicipalityCode;
    }
      
    function extractConsonants(str) {
      return str.replace(/[aeiou]/gi, '').toUpperCase();
    }

    function extractVowels(str) {
      return str.replace(/[^aeiou]/gi, '').toUpperCase();
    }
    
    function calculateLastName(lastName) {
      let lastNameCode;
      const lastNameConsonants = extractConsonants(lastName).slice(0, 3);
      const lastNameVowels = extractVowels(lastName).slice(0, 3);
      
      lastNameCode = lastNameConsonants;
      
      if (lastNameConsonants.length < 3) {
        lastNameCode = lastNameCode + lastNameVowels;
      }
      
      lastNameCode = lastNameCode + "XXX";
      
      return lastNameCode.slice(0, 3);
    }
    
    function calculateFirstName(firstName) {
      let firstNameCode;
      const firstNameConsonants = extractConsonants(firstName).slice(0, 4);
      const firstNameVowels = extractVowels(firstName).slice(0, 3);
      
      if (firstNameConsonants.length > 3) {
        firstNameCode = firstNameConsonants[0] + firstNameConsonants[2] + firstNameConsonants[3];
      } else {
        firstNameCode = firstNameConsonants;
      }
      
      if (firstNameConsonants.length < 3) {
        firstNameCode = firstNameCode + firstNameVowels;
      }
      
      firstNameCode = firstNameCode + "XXX";
      
      return firstNameCode.slice(0, 3);
    }

    function calculateBirthDate(date, sex) {
      const year = date.getFullYear().toString().slice(2);
      const month = "ABCDEHLMPRST"[(date.getMonth())];
      const daysex = sex === 'male' ? date.getDate() : date.getDate() + 40;

      return year + month + (daysex < 10 ? "0" + daysex : daysex);
    }
    
    function calculateControlCharacter(str) {
      const oddValues = { 0: 1, 1: 0, 2: 5, 3: 7, 4: 9, 5: 13, 6: 15, 7: 17, 8: 19, 9: 21, 
        A: 1, B: 0, C: 5, D: 7, E: 9, F: 13, G: 15, H: 17, I: 19, J: 21, K: 2, L: 4, M: 18, N: 20, 
        O: 11, P: 3, Q: 6, R: 8, S: 12, T: 14, U: 16, V: 10, W: 22, X: 25, Y: 24, Z: 23 };
      const evenValues = { 0: 0, 1: 1, 2: 2, 3: 3, 4: 4, 5: 5, 6: 6, 7: 7, 8: 8, 9: 9, 
        A: 0, B: 1, C: 2, D: 3, E: 4, F: 5, G: 6, H: 7, I: 8, J: 9, K: 10, L: 11, M: 12, N: 13, 
        O: 14, P: 15, Q: 16, R: 17, S: 18, T: 19, U: 20, V: 21, W: 22, X: 23, Y: 24, Z: 25 };
      
      let val = 0
      for (let i = 0; i < 15; i = i + 1) {
        const c = str[i]
        val += i % 2 !== 0 ? evenValues[c] : oddValues[c]
      }
      val = val % 26
      
      return 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.charAt(val)
    }

    const lastNameCode = calculateLastName(lastName);
    const firstNameCode = calculateFirstName(firstName);
    const birthData = calculateBirthDate(birthDate, sex);
    
    let taxCode = (lastNameCode + firstNameCode + birthData + municipalityCode).toUpperCase();
    taxCode = taxCode + calculateControlCharacter(taxCode);
      
    document.getElementById("cf-result").innerText = taxCode;
  }
  
  const inputs = document.querySelectorAll("#form-cf input");
  inputs.forEach(input => {
    input.addEventListener("input", function(event) {
      event.preventDefault();
      calculateTaxCode();
    })
  });
  
  document.getElementById("municipalityCode").addEventListener("input", (event) => {
    selectedMunicipalityCode = "";
    const inputValue = event.target.value;
    const matches = findMunicipalityCode(inputValue);
    updateSuggestions(matches);
  });

  document.addEventListener("click", (event) => {
    if (!event.target.closest("#municipalityCode")) {
      document.getElementById("municipalitySuggestions").style.display = "none";
    }
  });
  
  document.addEventListener("DOMContentLoaded", function() {
    const municipalityCode = document.getElementById("municipalityCode");
  
    loadMunicipalityData().then(() => {
      calculateTaxCode();
    });
    
    function clearOnFirstInput(event) {
      municipalityCode.value = "";
      municipalityCode.removeEventListener("input", clearOnFirstInput);
    }

    function resetListener() {
      municipalityCode.addEventListener("input", clearOnFirstInput);
    }
    
    municipalityCode.addEventListener("input", clearOnFirstInput);
    municipalityCode.addEventListener("blur", resetListener);
  });
</script>

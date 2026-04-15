# layer-gpu-pressure-test

## reference
### k3s
```bash
for p in 60 80 100 120; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30080/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":256}"
  ' | sort | uniq -c
done
```

### gateway-inference

```bash
# dev
for p in 20 40 60; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30180/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":256}"
  ' | sort | uniq -c
done

# prod small reqeust
for p in 20 40 60 80 100 120; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30380/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":56}"
  ' | sort | uniq -c
done


# prod large reqeust
for p in 20 40 60; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30380/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":256}"
  ' | sort | uniq -c
done

# prod mixed reqeust
SMALL='{"model":"Qwen/Qwen2.5-7B-Instruct","messages":[{"role":"user","content":"introduce new york city"}],"max_tokens":64}'
LARGE='{"model":"Qwen/Qwen2.5-7B-Instruct","messages":[{"role":"user","content":"Write a detailed 8-section travel guide for New York City including history, neighborhoods, transportation, food, attractions, itinerary, budget tips, and safety advice."}],"max_tokens":512}'

for p in 40 60 80; do
  echo "=== mixed random concurrency $p ==="

  seq 1 200 | xargs -I{} -P $p bash -c '
    r=$((RANDOM % 4))
    if [ $r -eq 0 ]; then
      TYPE="large"
      DATA='"'"$LARGE"'"'
    else
      TYPE="small"
      DATA='"'"$SMALL"'"'
    fi

    code=$(curl -s -o /dev/null -w "%{http_code}" http://192.168.86.179:30380/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "$DATA")

    echo "$TYPE $code"
  ' | sort | uniq -c

done
```

## gate-embed
```bash
# small 
for p in 20 60 100 140 180; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:8011/v1/embeddings \
      -H "Content-Type: application/json" \
      -H "X-Internal-Key: 1234" \
      -H "X-Request-Id: request_id_1" \
      -H "X-Session-Id: session_id_1" \
      -d "{\"model\":\"BAAI/bge-m3\",\"input\":\"New York City comprises 5 boroughs sitting where the Hudson River meets the Atlantic Ocean. At its core is Manhattan, a densely populated borough that’s among the world’s major commercial, financial and cultural centers. Its iconic sites include skyscrapers such as the Empire State Building and sprawling Central Park. Broadway theater is staged in neon-lit Times Square\"}"
  ' | sort | uniq -c
done

# large 
for p in 20 60 100 140 180; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:8011/v1/embeddings \
      -H "Content-Type: application/json" \
      -H "X-Internal-Key: 1234" \
      -H "X-Request-Id: request_id_1" \
      -H "X-Session-Id: session_id_1" \
      -d "{\"model\":\"BAAI/bge-m3\",\"input\":\"New York City is one of the most dynamic and influential urban centers in the world, known for its diversity, cultural significance, and economic power. The city comprises five boroughs: Manhattan, Brooklyn, Queens, the Bronx, and Staten Island. Each borough has its own unique identity, history, and role within the broader metropolitan area.

Manhattan is often considered the heart of the city, home to iconic landmarks such as Times Square, Central Park, Wall Street, and the Empire State Building. It is a global hub for finance, media, entertainment, and technology. The skyline of Manhattan is one of the most recognizable in the world, symbolizing ambition and innovation.

Brooklyn, located across the East River, has evolved into a cultural hotspot known for its artistic communities, diverse neighborhoods, and vibrant food scene. Areas like Williamsburg and DUMBO have become centers for startups, galleries, and creative industries. Brooklyn also boasts beautiful parks and waterfront views.

Queens is the largest borough by area and one of the most ethnically diverse places on Earth. It is home to neighborhoods representing cultures from around the globe, offering a wide range of cuisines and traditions. John F. Kennedy International Airport and LaGuardia Airport are both located in Queens, making it a key transportation hub.

The Bronx is known for its rich history and contributions to music and sports. It is the birthplace of hip-hop and home to Yankee Stadium. The borough also features the Bronx Zoo and the New York Botanical Garden, providing important green spaces for residents and visitors alike.

Staten Island, connected to Manhattan by the Staten Island Ferry, offers a quieter, more suburban atmosphere compared to the other boroughs. It provides scenic views of the Statue of Liberty and serves as a residential area for many New Yorkers.

New York City's economy is one of the largest in the world, driven by sectors such as finance, technology, healthcare, education, and tourism. Wall Street plays a critical role in global financial markets, while Silicon Alley represents the city's growing technology sector.

The city's public transportation system, including subways, buses, and commuter rails, enables millions of people to move efficiently every day. Despite its complexity, it is one of the most extensive transit systems globally.

Culturally, New York City is unmatched. It is home to Broadway theaters, world-class museums like the Metropolitan Museum of Art and the Museum of Modern Art, and countless music venues. The city's diversity fosters creativity and innovation across all forms of art.

Education is another cornerstone of the city, with institutions such as Columbia University, New York University, and the City University of New York system providing opportunities for higher learning and research.

Tourism is a major industry, with millions of visitors each year exploring landmarks such as the Statue of Liberty, Central Park, and the Brooklyn Bridge. The city's neighborhoods each offer unique experiences, from the historic streets of Harlem to the bustling avenues of Midtown.

Food in New York City reflects its diversity. From street vendors to Michelin-starred restaurants, the city offers cuisine from nearly every culture in the world. Pizza, bagels, and cheesecake are iconic staples.

The city's history dates back to its founding as New Amsterdam by Dutch settlers in the 17th century. Over time, it grew into a major port and gateway for immigrants arriving in the United States.

Today, New York City continues to evolve, balancing its historical roots with modern development. Skyscrapers continue to reshape the skyline, while communities work to preserve their cultural heritage.

The challenges facing the city include housing affordability, transportation infrastructure, and environmental sustainability. Efforts are ongoing to address climate change impacts, particularly rising sea levels and extreme weather events.

Despite these challenges, New York City remains a symbol of opportunity, resilience, and ambition. It attracts people from all over the world who come to pursue their dreams and contribute to its vibrant culture.

New York City's influence extends far beyond its borders. It serves as a global center for media, fashion, and finance, shaping trends and policies worldwide.

The city's parks and public spaces provide essential areas for recreation and relaxation. Central Park, in particular, is a landmark that offers a green oasis in the midst of urban density.

Innovation and entrepreneurship thrive in New York City, supported by a strong ecosystem of investors, incubators, and talent. Startups across various industries continue to emerge and grow.

The arts scene is continually evolving, with new galleries, performances, and creative expressions emerging throughout the city. Street art and public installations add to the city's dynamic visual landscape.

Community organizations and local initiatives play a vital role in supporting residents and addressing social issues. These efforts contribute to the city's resilience and adaptability.

As a global city, New York remains at the forefront of economic, cultural, and technological developments. Its ability to reinvent itself ensures its continued relevance in an ever-changing world.

New York City is not just a place; it is an experience defined by its energy,\"}"
  ' | sort | uniq -c
done
```



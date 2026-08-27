name: DH건설정보 월차 알림

on:
  schedule:
    - cron: '0 0 * * *'   # 매일 00:00 UTC = 09:00 KST
  workflow_dispatch: {}    # Actions 탭에서 수동 실행도 가능

permissions:
  contents: write

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run leave tracker
        env:
          BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
        run: |
          set -e
          STATE_FILE="state.json"
          if [ ! -f "$STATE_FILE" ]; then
            echo '{"hire_date":"2026-07-15","chat_id":6110934837,"last_notified_month_index":1,"used":0,"last_update_id":275110740,"probation_notified":[]}' > "$STATE_FILE"
          fi

          HIRE_DATE=$(jq -r '.hire_date' "$STATE_FILE")
          CHAT_ID=$(jq -r '.chat_id' "$STATE_FILE")
          LAST_IDX=$(jq -r '.last_notified_month_index' "$STATE_FILE")
          USED=$(jq -r '.used' "$STATE_FILE")
          LAST_UPDATE_ID=$(jq -r '.last_update_id' "$STATE_FILE")
          PROBATION_NOTIFIED=$(jq -r '(.probation_notified // []) | join(",")' "$STATE_FILE")

          TODAY=$(date -u +%Y-%m-%d)
          CHANGED=0

          send_msg() {
            curl -s "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
              --data-urlencode "chat_id=${CHAT_ID}" \
              --data-urlencode "text=$1" > /dev/null
          }

          # 1) 만근 월차 발생 체크 (입사일 기준 매월 15일, 1~11개월차)
          while [ "$LAST_IDX" -lt 11 ]; do
            NEXT_IDX=$((LAST_IDX + 1))
            ACCRUAL_DATE=$(date -u -d "${HIRE_DATE} +${NEXT_IDX} months" +%Y-%m-%d)
            if [[ "$ACCRUAL_DATE" > "$TODAY" ]]; then
              break
            fi
            EXPIRY_DATE=$(date -u -d "${ACCRUAL_DATE} +1 year -1 day" +%Y-%m-%d)
            LAST_IDX=$NEXT_IDX
            AVAILABLE=$((LAST_IDX - USED))
            MSG="[DH건설정보] 월차 발생 🎉%0A%0A${NEXT_IDX}개월차 만근 월차 1일 발생%0A발생일: ${ACCRUAL_DATE}%0A사용기한: ${EXPIRY_DATE}%0A사용 가능 잔여: ${AVAILABLE}일"
            send_msg "$MSG"
            CHANGED=1
          done

          # 2) 수습 종료 디데이 카운트다운 (2026-10-17 종료 기준: D-90~D-10은 10일 단위, 이후 D-7/D-3/D-1/D-Day)
          PROBATION_END="2026-10-17"

          check_probation_milestone() {
            local label="$1" mdate="$2"
            if [[ "$mdate" > "$TODAY" ]]; then
              return
            fi
            if [[ ",${PROBATION_NOTIFIED}," == *",${label},"* ]]; then
              return
            fi
            local body
            if [ "$label" == "D-Day" ]; then
              body="오늘(${PROBATION_END}) 수습 기간이 종료됩니다."
            else
              body="수습 종료(${PROBATION_END})까지 ${label#D-}일 남았습니다."
            fi
            send_msg "[DH건설정보] 수습 종료 ${label}%0A%0A${body}"
            PROBATION_NOTIFIED="${PROBATION_NOTIFIED:+${PROBATION_NOTIFIED},}${label}"
            CHANGED=1
          }

          for N in 90 80 70 60 50 40 30 20 10; do
            check_probation_milestone "D-${N}" "$(date -u -d "${PROBATION_END} -${N} days" +%Y-%m-%d)"
          done

          check_probation_milestone "D-7" "$(date -u -d "${PROBATION_END} -7 days" +%Y-%m-%d)"
          check_probation_milestone "D-3" "$(date -u -d "${PROBATION_END} -3 days" +%Y-%m-%d)"
          check_probation_milestone "D-1" "$(date -u -d "${PROBATION_END} -1 days" +%Y-%m-%d)"
          check_probation_milestone "D-Day" "$PROBATION_END"

          # 3) 월차 사용 신고 체크 (텔레그램으로 받은 새 메시지 처리)
          OFFSET=$((LAST_UPDATE_ID + 1))
          UPDATES=$(curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getUpdates" --data-urlencode "offset=${OFFSET}")
          COUNT=$(echo "$UPDATES" | jq '.result | length')

          if [ "$COUNT" -gt 0 ]; then
            for i in $(seq 0 $((COUNT - 1))); do
              UPD_ID=$(echo "$UPDATES" | jq -r ".result[$i].update_id")
              MSG_CHAT_ID=$(echo "$UPDATES" | jq -r ".result[$i].message.chat.id // empty")
              TEXT=$(echo "$UPDATES" | jq -r ".result[$i].message.text // empty")

              if [ "$MSG_CHAT_ID" == "$CHAT_ID" ]; then
                AVAILABLE=$((LAST_IDX - USED))
                if echo "$TEXT" | grep -qE "사용|썼|씀|소진"; then
                  if [ "$AVAILABLE" -gt 0 ]; then
                    USED=$((USED + 1))
                    NEW_AVAIL=$((LAST_IDX - USED))
                    send_msg "[DH건설정보] 월차 1일 사용 처리 완료%0A남은 월차: ${NEW_AVAIL}일"
                  else
                    send_msg "[DH건설정보] 사용 가능한 월차가 없습니다."
                  fi
                else
                  send_msg "[DH건설정보] '월차 사용'이라고 보내주시면 1일 차감해 드려요. 현재 사용 가능: ${AVAILABLE}일"
                fi
              fi

              LAST_UPDATE_ID=$UPD_ID
              CHANGED=1
            done
          fi

          if [ -z "$PROBATION_NOTIFIED" ]; then
            PROBATION_JSON="[]"
          else
            PROBATION_JSON=$(printf '%s' "$PROBATION_NOTIFIED" | jq -R 'split(",")')
          fi

          jq -n --arg hire "$HIRE_DATE" --argjson chat "$CHAT_ID" --argjson idx "$LAST_IDX" --argjson used "$USED" --argjson upd "$LAST_UPDATE_ID" --argjson pn "$PROBATION_JSON" \
            '{hire_date:$hire, chat_id:$chat, last_notified_month_index:$idx, used:$used, last_update_id:$upd, probation_notified:$pn}' > "$STATE_FILE"

          echo "CHANGED=$CHANGED" >> "$GITHUB_ENV"

      - name: Commit state
        if: env.CHANGED == '1'
        run: |
          git config user.name "leave-tracker-bot"
          git config user.email "actions@users.noreply.github.com"
          git add state.json
          git commit -m "update leave state" || echo "nothing to commit"
          git push

FROM alpine:3.14.0


RUN apk add curl
RUN curl  -ko argocd https://argocd.control.clickthebutton.click/download/argocd-linux-amd64
RUN chmod +x argocd && mv ./argocd /bin/

RUN argocd

ENTRYPOINT ["cat"]
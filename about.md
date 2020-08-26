---
layout: page
title: About
permalink: /about/
---

Hi there, I'm Blat Blatnik - a student of artificial intelligence and a huge programming geek. I decided to create [`computerBear`]({{"/" | relative_url}}) to share some of my coding discoveries and frustrations. 

I'm particularly interested in game development and low-level programming in general, and I'm working on my own game right now.

## Snake

Need a break? How about a game of Snake?

<noscript>
  <p style="text-align:center;">
    You have to enable javascript to play Snake.
  </p>
</noscript>
<canvas id="canvas" tabindex="-1" style="height: 50%;">Canvas not supported</canvas>

<script type="text/javascript">
var e=document.getElementById("canvas"),h=e.getContext("2d",{alpha:!1}),k=e.getBoundingClientRect();e.setAttribute("width",k.width);e.setAttribute("height",k.height);e.focus();var l=40,n=20,p=Array(4),r=0,t=0,u=0,v=0,w=20,x=10,y=Array(1600);y[0]=w;y[1]=x;y[2]=w;y[3]=x;y[4]=w;y[5]=x;
for(var z=6,B=2,C=0,D="score: 0",ba=aa(),F=0,G=Array(1600),H=[],I=G.length,J=[],K=Array(101),L=[{r:255,b:0,a:0},{r:255,b:136,a:0},{r:97,b:46,a:0}],M=0;20>M;++M)for(var N=0;40>N;++N){var ca=2*(40*M+N);G[ca+0]=N;G[ca+1]=M}for(var O=0;O<K.length;++O){var P,Q=Math.min(Math.max(O/(K.length-1)*L.length,0),L.length-1),R=L[Math.floor(Q)],S=L[Math.ceil(Q)],da=Q-Math.floor(Q);P={r:T(R.r,S.r),b:T(R.b,S.b),a:T(R.a,S.a)};K[O]="rgb("+P.r+","+P.b+","+P.a+")"}U();e.addEventListener("keydown",ea);
e.addEventListener("focusout",fa);V();
function V(){var a=aa(),b=(a-ba)/1E3;ba=a;.018>=b?b=1/60:.019<=b&&.021>=b?b=.02:.034<=b&&(b=1/30);if(0==B){F+=b;1==t?u-=20*b:2==t?u+=20*b:3==t?v-=20*b:4==t&&(v+=20*b);if(b=0!=t&&(1<=Math.abs(u)||1<=Math.abs(v))){for(a=z-2;2<=a;a-=2)y[a+0]=y[a-2],y[a+1]=y[a-1];-1>=u?(y[0]--,u++):1<=u?(y[0]++,u--):-1>=v?(y[1]--,v++):1<=v&&(y[1]++,v--);0>y[0]&&(y[0]=39);40<=y[0]&&(y[0]=0);0>y[1]&&(y[1]=19);20<=y[1]&&(y[1]=0);for(a=0;a<H.length;a+=3){var c=H[a+0],g=H[a+1],d=H[a+2];if(y[0]==c&&y[1]==g){console.log("eat fruit. "+
(H.length-3)/3+" remaining");C+=Math.max(1,Math.ceil((1-Math.min(Math.max((F-d)/(5+z/20),0),1))*z/2));D="score: "+Math.ceil(C);J.push(c,g,F);H.splice(a,3);0==H.length&&(console.log("trying to make next fruit"),U());break}}c=y[z-2];g=y[z-1];for(a=0;a<J.length;a+=3)d=J[a+1],J[a+0]==c&&d==g&&(y[z++]=c,y[z++]=g,J.splice(a,3),a-=3);if(6<z)for(a=2;a<z;a+=2)if(y[0]==y[a+0]&&y[1]==y[a+1]){B=3;break}}if((b||0==t)&&0<r){b=p[0];for(a=1;a<r;++a)p[a-1]=p[a];--r;ha(b,t)||(t=b,v=u=0)}}a=e.width;b=e.height;l=a/40;
n=b/20;h.globalCompositeOperation="source-over";h.fillStyle="white";h.fillRect(0,0,a,b);for(c=0;c<H.length;c+=3){g=H[c+0];d=H[c+1];var f=Math.min(Math.max((F-H[c+2])/(5+z/20),0),1);h.fillStyle=K[Math.floor(100*f)];f=1-ia(f)/3;var m=(1-f)/2;W(g+m,d+m,f,f,.25*f,!1)}h.fillStyle="rgb(50,50,50)";c=1+z/2;for(g=0;g<J.length;g+=3)d=.8+.3*(1-20*(F-J[g+2])/c),f=(1-d)/2,W(J[g+0]+f,J[g+1]+f,d,d,d/4,!1);c=y[z-2];g=y[z-1];d=y[z-4]-c;f=y[z-3]-g;d=0<d&&20>d||-20>d?2:0>d&&20>-d||20<d?1:0<f&&10>f||-10>f?4:0>f&&10>
-f||10<f?3:0;f=Math.abs(u)+Math.abs(v);1==d?c-=f:2==d?c+=f:3==d?g-=f:4==d&&(g+=f);h.fillStyle="black";d=y[0];f=y[1];X(d+u,f+v,d,f);for(m=2;m<z-2;m+=2){var q=y[m+0],A=y[m+1];X(d,f,q,A);d=q;f=A}X(c,g,d,f);if(0==B||1==B)h.font="400 20px Serif",h.textAlign="center",h.fillStyle="rgb(120,120,120)",h.fillText(D,a/2,40);0==B?window.requestAnimationFrame(V):(2!=B&&(h.globalCompositeOperation="multiply",h.fillStyle="rgb(200,200,200)",h.fillRect(0,0,a,b),h.globalCompositeOperation="source-over"),h.font="400 32px Serif",
h.textAlign="center",h.fillStyle="rgb(120,120,120)",2==B?h.fillText("Move to Start",a/2,b/4):1==B?(h.fillText("Paused",a/2,b/2),h.font="400 20px Serif",h.fillText("Press space to resume",a/2,b/2+40)):3==B&&(h.fillText("Game Over!",a/2,b/4),h.fillText(D,a/2,b/4+40),h.font="400 20px Serif",h.fillText("Press space to try again",a/2,b/4+80)))}
function X(a,b,c,g){var d=Math.min(a,c);a=Math.max(a,c);c=Math.min(b,g);b=Math.max(b,g);g=a-d;var f=b-c,m=g+1,q=f+1;1<g?(Y(d-1,c,2,1),Y(a,b,2,1)):1<f?(Y(d,c-1,1,2),Y(a,b,1,2)):(Y(d,c,m,q),0>d?Y(40+d,c,m,q):39<a?Y(d-40,c,m,q):0>c?Y(d,20+c,m,q):19<b&&Y(d,c-20,m,q))}function Y(a,b,c,g){W(a+.1,b+.1,c-.2,g-.2,.5,!0)}
function W(a,b,c,g,d,f){a*=l;b*=n;c*=l;g*=n;d*=Math.min(l,n);f?(a=Math.ceil(a),b=Math.ceil(b),c=Math.floor(c),g=Math.floor(g)):(a+=.5,b+=.5);h.beginPath();h.moveTo(a+d,b);h.lineTo(a+c-d,b);h.quadraticCurveTo(a+c,b,a+c,b+d);h.lineTo(a+c,b+g-d);h.quadraticCurveTo(a+c,b+g,a+c-d,b+g);h.lineTo(a+d,b+g);h.quadraticCurveTo(a,b+g,a,b+g-d);h.lineTo(a,b+d);h.quadraticCurveTo(a,b,a+d,b);h.closePath();h.fill()}
function U(){if(1600>z){var a;do{if(I>=G.length){for(a=G.length-2;2<=a;a-=2){var b=2*Math.floor(Math.random()*a/2);var c=G[a+0],g=G[a+1];G[a+0]=G[b+0];G[a+1]=G[b+1];G[b+0]=c;G[b+1]=g}I=0}a=G[I++];b=G[I++]}while(Z(a,b));console.log("made fruit at ("+a+", "+b+")");H.push(a,b,F);if((z-20)/2E3>Math.random()){c=Math.random();if(.25>c&&39>a&&19>b){var d=a+1;var f=b;var m=a;var q=b+1;var A=a+1;var E=b+1}else.5>c&&37>a?(d=a+1,f=b,m=a+2,q=b,A=a+3,E=b):.75>c&&17>b?(d=a,f=b+1,m=a,q=b+2,A=a,E=b+3):38>a&&19>b&&
(d=a+1,f=b,m=a+1,q=b+1,A=a+2,E=b+1);Z(d,f)||Z(m,q)||Z(A,E)||H.push(d,f,F,m,q,F,A,E,F)}}}function fa(){0==B&&(B=1)}
function ea(a){var b=B;if(32==a.keyCode||13==a.keyCode||27==a.keyCode)a.preventDefault(),0==B?B=1:1==B?B=0:3==B&&(t=r=0,w=20,x=10,z=6,y[0]=w,y[1]=x,y[2]=w,y[3]=x,y[4]=w,y[5]=x,v=u=0,B=2,C=0,D="score: 0",F=0,J.length=0,H.length=0,U(),window.requestAnimationFrame(V));else{a.preventDefault();var c=0;switch(a.keyCode){case 37:case 65:case 100:c=1;break;case 38:case 87:case 104:c=3;break;case 39:case 68:case 102:c=2;break;case 40:case 83:case 101:c=4}if(0!=c&&2==B||0==B)r<p.length&&(ha(c,0==r?t:p[r-1])||
(p[r++]=c)),2==B&&(B=0)}0==B&&0!=b&&window.requestAnimationFrame(V)}function Z(a,b){for(var c=0;c<z;c+=2)if(y[c+0]==a&&y[c+1]==b)return!0;return!1}function ha(a,b){return 0==a||0==b?!1:(1==a||2==a)==(1==b||2==b)||(3==a||4==a)==(3==b||4==b)}function aa(){return window.performance.now?window.performance.now():window.performance.webkitNow?window.performance.webkitNow():(new Date).getTime()}function T(a,b){return a+(b-a)*ia(da)}function ia(a){a=Math.min(Math.max(a-0,0),1);return a*a*(3-2*a)};
</script>
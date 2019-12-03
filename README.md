# Instruções
# As linhas de codigo abaixo deverão ser executadas no Google Colab. Primeiro salve os arquivos no seu Google Drive. Depois, crie um diretório. Obs.: os "paths" deverão ser os mesmos criados e não os indicados nas linhas de código abaixo.


# Etapa 1 - Importando as bibliotecas
import cv2
import numpy as np
import time
import pandas as pd
import matplotlib.pyplot as plt
from google.colab.patches import cv2_imshow
import zipfile

cv2.__version__

%tensorflow_version 2.x
import tensorflow
tensorflow.__version__

# Etapa 2 - Conectando com o Drive e acessando os arquivos
from google.colab import drive
drive.mount('/content/gdrive')

path = "/content/gdrive/My Drive/Videos.zip"
zip_object = zipfile.ZipFile(file=path, mode="r")
zip_object.extractall("./")

# Etapa 3 - Carregando o modelo
from tensorflow.keras.models import load_model

diretorio = 'gdrive/My Drive/Cursos/Deteccao_Expressoes_Faciais/'

model = load_model(diretorio + "modelo_02_expressoes.h5")

# Etapa 4 - Carregando o vídeo
arquivo_video = diretorio + "testes/video_teste04.mp4"
cap = cv2.VideoCapture(arquivo_video)

conectado, video = cap.read()
print(video.shape)

# Etapa 5 - Redimensionando o tamanho (opcional)
# Recomendado quando o tamanho do vídeo é muito grande. Se o vídeo tiver a resolução muito alta então pode demorar muito o processamento.
redimensionar = True 
# deixe True para reduzir o tamanho do vídeo salvo caso este supere a largura máxima que vamos especificar abaixo.
# para manter o tamanho original deixe False 
largura_maxima = 600  # pixels. define o tamanho da largura (máxima) do vídeo a ser salvo. a altura será proporcional e é definida nos calculos abaixo

# se redimensionar = True então o video que será salvo terá seu tamanho em pixels reduzido SE for maior que a largura_maxima

if (redimensionar and video.shape[1]>largura_maxima):
  # precisamos deixar a largura e altura proporcionais (mantendo a proporção do vídeo original) para que a imagem não fique com aparência esticada
  proporcao = video.shape[1] / video.shape[0]
  # para isso devemos calcular a proporção (largura/altura) e usaremos esse valor para calcular a altura (com base na largura que definimos acima) 
  video_largura = largura_maxima
  video_altura = int(video_largura / proporcao)
else:
  video_largura = video.shape[1]
  video_altura = video.shape[0]

# se redimensionar = False então os valores da largura e altura permanecerão os mesmos do vídeo original  

# Etapa 6 - Definindo as configurações do vídeo
# nome do arquivo de vídeo que será salvo
nome_arquivo = diretorio+'resultado_video_teste04.avi' 

# definição do codec
fourcc = cv2.VideoWriter_fourcc(*'XVID') 
# FourCC é um código de 4 bytes usado para especificar o codec de vídeo. A lista de códigos disponíveis pode ser encontrada no site fourcc.org
# Codecs mais usados: XVID, MP4V, MJPG, DIVX, X264... 
# Por exemplo, para salvar em formato mp4 utiliza-se o codec mp4v (o nome do arquivo também precisa possuir a extensão .mp4)
# fourcc = cv2.VideoWriter_fourcc(*'mp4v') 

# FPS - frames por segundo
fps = 24
# se quiser deixar o video um pouco mais lento pode diminuir o numero de frames por segundo para 20

saida_video = cv2.VideoWriter(nome_arquivo, fourcc, fps, (video_largura, video_altura))

# Etapa 7 - Processamento do vídeo e gravação do resultado
from tensorflow.keras.preprocessing.image import img_to_array

haarcascade_faces = diretorio + 'haarcascade_frontalface_alt.xml' # arquivo haarcascade

# define os tamanhos para as fontes
fonte_pequena, fonte_media = 0.4, 0.7

fonte = cv2.FONT_HERSHEY_SIMPLEX

expressoes = ["Raiva", "Nojo", "Medo", "Feliz", "Triste", "Surpreso", "Neutro"] 

while (cv2.waitKey(1) < 0):
    conectado, frame = cap.read()
    if not conectado:
        break  # se ocorreu um problema ao carregar a imagem então interrompe o programa

    t = time.time() # tempo atual, antes de iniciar (vamos utilizar para calcular quanto tempo levou para executar as operações)
    
    # frame_video = np.copy(frame) # faz uma copia do frame do video

    if redimensionar: # se redimensionar = True então redimensiona o frame para os novos tamanhos
      frame = cv2.resize(frame, (video_largura, video_altura)) 

    face_cascade = cv2.CascadeClassifier(haarcascade_faces)
    cinza = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY) # converte pra grayscale
    faces = face_cascade.detectMultiScale(cinza,scaleFactor=1.2, minNeighbors=5,minSize=(30,30))

    if len(faces) > 0:
        for (x, y, w, h) in faces:

            frame = cv2.rectangle(frame,(x,y),(x+w,y+h+10),(255,50,50),2) # desenha retângulo ao redor da face

            roi = cinza[y:y + h, x:x + w]      # extrai apenas a região de interesse (ROI) que é onde contém o rosto 
            roi = cv2.resize(roi, (48, 48))    # antes de passar pra rede neural redimensiona para o tamanho das imagens de treinamento
            roi = roi.astype("float") / 255.0  # normaliza
            roi = img_to_array(roi)            # converte para array para que a rede possa processar
            roi = np.expand_dims(roi, axis=0)  # muda o shape da array

            # faz a predição - calcula as probabilidades
            result = model.predict(roi)[0]
            print(result)
                
            if result is not None:
                resultado = np.argmax(result) # encontra a emoção com maior probabilidade
                cv2.putText(frame,expressoes[resultado],(x,y-10), fonte, fonte_media,(255,255,255),1,cv2.LINE_AA) # escreve a emoção acima do rosto

    # tempo processado = tempo atual (time.time()) - tempo inicial (t)
    cv2.putText(frame, " frame processado em {:.2f} segundos".format(time.time() - t), (20, video_altura-20), fonte, fonte_pequena, (250, 250, 250), 0, lineType=cv2.LINE_AA)

    cv2_imshow(frame) 
    saida_video.write(frame) # grava o frame atual
    
print("Terminou")

saida_video.release()

cv2.destroyAllWindows()


















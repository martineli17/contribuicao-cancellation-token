# CancellationToken
### Contextualizando
Imagine o seguinte cenário: 
1. Uma pessoa vai até um restaurante - restaurante está cheio.
2. Faz um pedido, o garçom anota e entrega aos chefs de cozinha. 
3. Os chefs, depois de um tempo razoável, disponibiliza o pedido para entrega. 
4. Então, o garçom pega a entrega e se direciona à mesa em que o cliente estava, mas o cliente já tinha ido embora.

Em suma: os chefs de cozinha gastaram tempo e esforço e o restaurante gastou ingredientes em um pedido que não era mais necessário.

Essa contextualização foi para dizer que na área de desenvolvimento de aplicações também ocorre essa situação.
Muitas das vezes, quando construímos nossas aplicações, há possilidades de quem estiver consumindo aquele serviço, desistir da tarefa/requisição antes mesmo dela ser finalizada.
Nós, como desenvolvedores, devemos desenvolver softwares pensando nessa possibilidade: não podemos gastar mais recursos com algo que não será mais necessário, há outros processos aguardando. 

Mas, aí vem a pergunta: como podemos implementar isso em nosso software?
Em .NET, pelo menos, tem uma solução bem interessante e relativamente simples: o CancellationToken.
### Utilização em API
No contexto de utilização em uma API, podemos injetar o CancellationToken diretamente no endpoint que desejamos. Ele será referenciado ao request enviado.
Quando o CancellationToken for cancelado, essa mudança de evento será enviada para todos os restantes dos processos que o utilizam (se trata de um tipo complexo, então a passagem é feita por referência).
```c#
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

namespace TesteCancellationToken.Controllers
{
    [ApiController]
    public class CancellationTokenController : ControllerBase
    {
        private readonly ILogger _logger;
        public CancellationTokenController(ILogger<CancellationTokenController> logger)
        {
            _logger = logger;
        }

        [HttpGet("cancellation-token")]
        public async Task Teste(CancellationToken cancellationToken) // recebendo o token automaticamente referente ao request
        {
            var returns = await Execute(cancellationToken);
            //registra uma ação que executurá após o cancelamento do request (pode inserir mais de um Register)
            cancellationToken.Register(() => LoggerCancel(returns), true);
            cancellationToken.Register(() => LoggerCancelExtra(), true);
        }

        private Task<List<string>> Execute(CancellationToken cancellationToken)
        {
            var returns = new List<string>();
            int countTasks = 0;
            //enquanto o request não for cancelado, ele irá executar o método
            while (!cancellationToken.IsCancellationRequested)
            {
                Thread.Sleep(1000);
                countTasks++;
                var taskName = $"Task [{countTasks}]";
                if (cancellationToken.IsCancellationRequested)
                {
                    returns.Add($"{taskName} cancelada");
                    break;
                }
                _logger.LogInformation($"{taskName} iniciada");
                returns.Add($"{taskName} iniciada");
            }
            return Task.FromResult(returns);
        }
        
        private void LoggerCancel(List<string> listParam) =>
            listParam.ForEach(x => _logger.LogInformation($"{x} e cancelada."));
            
        private void LoggerCancelExtra() => _logger.LogInformation("Logger extra!");
    }
}
```
Resumindo: enquanto a requisição não for cancelada -  ``` while (!cancellationToken.IsCancellationRequested) ``` - continuará executando o código do while. E ao cancelar, executrá os métodos  ``` Register ``` do próprio CancellationToken.

### Vamos ver na prática?
Enviando o request - o request está em aberto.

![image](https://user-images.githubusercontent.com/50757499/116161390-0c9aac80-a6ca-11eb-9ce6-29f38c8e3139.png)

Enquanto não cancelamos o request, a execução do código ocorre normalmente. A partir do momento que cancelamos o request - o cliente desistiu da requisição - paramos o nosso processamento, já que não faz sentido continuar sendo que o cliente não vai querer mais a resposta.

![image](https://user-images.githubusercontent.com/50757499/116161977-330d1780-a6cb-11eb-9dba-62499a494b30.png)

### Então, consigo utilizar somente em API?
Não. Você pode criar o seu próprio CancellationToken em um processo principal, por exemplo, e repassá-lo aos demais processos que o principal executa.
Segue exemplo que você pode utilizar: 
```c#
 public CancellationTokenSource CriarCancellationTokenSource()
 {
     var cancellationTokenSource = new CancellationTokenSource();
     var cancellationToken = cancellationTokenSource.Token;
     if (alguma condição) 
             cancellationTokenSource.Cancel(); //cancela o cancellationToken
     return cancellationTokenSource;
 }
```

### Conclusão
Analise cuidadosamente o seu cenário e sua necessidade e, se possível, utilize o CancellationToken.
Assim, após o 'desistência' da finalização do processo, conseguimos cancelar a execução dos demais processos que ainda não foram executados.
